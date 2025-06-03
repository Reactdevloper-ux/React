protected override Task ExecuteAsync(CancellationToken stoppingToken)
{
    var connectionString = _config["ServiceBus:ConnectionString"];

    var topicSubscriptionPairs = new[]
    {
        new { Topic = _config["ServiceBus:TopicBegin"], Subscription = _config["ServiceBus:SubscriptionBegin"] },
        new { Topic = _config["ServiceBus:TopicStatus"], Subscription = _config["ServiceBus:SubscriptionStatus"] }
    };

    foreach (var pair in topicSubscriptionPairs)
    {
        var client = new SubscriptionClient(connectionString, pair.Topic, pair.Subscription);
        _subscriptionClients.Add(client);

        client.RegisterMessageHandler(async (message, token) =>
        {
            var body = Encoding.UTF8.GetString(message.Body);
            await _storageService.SaveMessageAsync(body);
            await _hubContext.Clients.All.SendAsync("ReceiveMessage", body);
            await client.CompleteAsync(message.SystemProperties.LockToken);
        },
        new MessageHandlerOptions(args =>
        {
            Console.WriteLine($"Error: {args.Exception}");
            return Task.CompletedTask;
        })
        {
            MaxConcurrentCalls = 1,
            AutoComplete = false
        });
    }

    return Task.CompletedTask;
}

"ServiceBus": {
  "ConnectionString": "Endpoint=sb://dbsm-n-1-dtlm-sb-1.servicebus.windows.net/;SharedAccessKeyName=...;SharedAccessKey=...",
  "TopicBegin": "publication-begin",
  "TopicStatus": "publication-status",
  "SubscriptionBegin": "subscription-begin",
  "SubscriptionStatus": "subscription-status"
}
using Backend.Data;
using Backend.Models;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.SignalR;
using Backend.Hubs;

namespace Backend.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class MessagesController : ControllerBase
    {
        private readonly AppDbContext _context;
        private readonly IHubContext<MessageHub> _hubContext;

        public MessagesController(AppDbContext context, IHubContext<MessageHub> hubContext)
        {
            _context = context;
            _hubContext = hubContext;
        }

        // GET: api/messages
        [HttpGet]
        public IActionResult GetMessages()
        {
            var messages = _context.Messages
                .OrderByDescending(m => m.ReceivedAt)
                .Take(100)
                .ToList();

            return Ok(messages);
        }

        // POST: api/messages/test
        [HttpPost("test")]
        public async Task<IActionResult> SendTestMessage([FromBody] string message)
        {
            var entity = new MessageEntity
            {
                Content = message,
                ReceivedAt = DateTime.UtcNow
            };

            _context.Messages.Add(entity);
            await _context.SaveChangesAsync();

            await _hubContext.Clients.All.SendAsync("ReceiveMessage", message);

            return Ok(new { sent = message });
        }
    }
}


/Backend
│
├── Controllers/
├── Hubs/
│   └── MessageHub.cs
├── Services/
│   ├── ServiceBusBackgroundService.cs
│   └── IMessageStorageService.cs
├── Data/
│   └── AppDbContext.cs
├── Models/
│   └── MessageEntity.cs
├── Program.cs
├── appsettings.json
└── Backend.csproj

Models/MessageEntity.cs
public class MessageEntity
{
    public int Id { get; set; }
    public string Content { get; set; }
    public DateTime ReceivedAt { get; set; }
}
Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using Backend.Models;

namespace Backend.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

        public DbSet<MessageEntity> Messages { get; set; }
    }
}
Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using Backend.Models;

namespace Backend.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

        public DbSet<MessageEntity> Messages { get; set; }
    }
}

Services/IMessageStorageService.cs
using Backend.Models;

namespace Backend.Services
{
    public interface IMessageStorageService
    {
        Task SaveMessageAsync(string message);
    }

    public class MessageStorageService : IMessageStorageService
    {
        private readonly AppDbContext _context;

        public MessageStorageService(AppDbContext context)
        {
            _context = context;
        }

        public async Task SaveMessageAsync(string message)
        {
            var entity = new MessageEntity
            {
                Content = message,
                ReceivedAt = DateTime.UtcNow
            };

            _context.Messages.Add(entity);
            await _context.SaveChangesAsync();
        }
    }
}

Services/ServiceBusBackgroundService.cs
using Microsoft.Azure.ServiceBus;
using Microsoft.AspNetCore.SignalR;
using System.Text;
using Backend.Hubs;

namespace Backend.Services
{
    public class ServiceBusBackgroundService : BackgroundService
    {
        private readonly IConfiguration _config;
        private readonly IHubContext<MessageHub> _hubContext;
        private readonly IMessageStorageService _storageService;

        private readonly List<ISubscriptionClient> _subscriptionClients = new();

        public ServiceBusBackgroundService(IConfiguration config, IHubContext<MessageHub> hubContext, IMessageStorageService storageService)
        {
            _config = config;
            _hubContext = hubContext;
            _storageService = storageService;
        }

        protected override Task ExecuteAsync(CancellationToken stoppingToken)
        {
            var connectionString = _config["ServiceBus:ConnectionString"];
            var topicName = _config["ServiceBus:TopicName"];
            var subscriptions = _config.GetSection("ServiceBus:Subscriptions").Get<string[]>();

            foreach (var sub in subscriptions)
            {
                var client = new SubscriptionClient(connectionString, topicName, sub);
                _subscriptionClients.Add(client);

                client.RegisterMessageHandler(async (message, token) =>
                {
                    var body = Encoding.UTF8.GetString(message.Body);

                    await _storageService.SaveMessageAsync(body);
                    await _hubContext.Clients.All.SendAsync("ReceiveMessage", body);
                    await client.CompleteAsync(message.SystemProperties.LockToken);

                }, new MessageHandlerOptions(args =>
                {
                    Console.WriteLine($"Error in subscription {sub}: {args.Exception}");
                    return Task.CompletedTask;
                })
                {
                    MaxConcurrentCalls = 1,
                    AutoComplete = false
                });
            }

            return Task.CompletedTask;
        }

        public override async Task StopAsync(CancellationToken cancellationToken)
        {
            foreach (var client in _subscriptionClients)
            {
                await client.CloseAsync();
            }

            await base.StopAsync(cancellationToken);
        }
    }
}

Hubs/MessageHub.cs
using Microsoft.AspNetCore.SignalR;

namespace Backend.Hubs
{
    public class MessageHub : Hub
    {
    }
}
appsettings.json
{
  "ServiceBus": {
    "ConnectionString": "your-service-bus-connection-string",
    "TopicName": "your-topic-name",
    "Subscriptions": [ "sub1", "sub2" ]
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MessagesDb;Trusted_Connection=True;TrustServerCertificate=True;"
  }
}

Program.cs
using Backend.Data;
using Backend.Hubs;
using Backend.Services;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddSignalR();

builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials()
              .WithOrigins("http://localhost:3000");
    });
});

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddScoped<IMessageStorageService, MessageStorageService>();
builder.Services.AddHostedService<ServiceBusBackgroundService>();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors();
app.UseAuthorization();

app.MapControllers();
app.MapHub<MessageHub>("/messageHub");

app.Run();

dotnet ef migrations add Init
dotnet ef database update




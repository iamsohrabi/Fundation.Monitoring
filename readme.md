# Fundation.Monitoring

کتابخانه‌ای برای یکپارچه‌کردن قابلیت‌های مانیتورینگ در سرویس‌های ASP.NET Core مبتنی بر .NET 10. هدف آن این است که هر سرویس با چند فراخوانی، health checkهای استاندارد، endpoint متریک‌های Prometheus و instrumentation مبتنی بر OpenTelemetry داشته باشد.

## قابلیت‌ها

- ثبت health checkهای خود برنامه و وابستگی‌هایی مانند PostgreSQL، MongoDB، RabbitMQ و EventStore از طریق پکیج‌های HealthChecks.
- endpointهای liveness و readiness برای orchestrationهایی مثل Kubernetes.
- health checkهای دسته‌بندی‌شده با tagهای `infra` و `database`.
- داشبورد HealthChecks UI با بررسی endpointها هر ۶۰ ثانیه.
- انتشار متریک‌های Prometheus در مسیر `/metrics`، شامل متریک‌های HTTP و gRPC.
- ثبت traceهای ASP.NET Core و `HttpClient` و متریک‌های ASP.NET Core، `HttpClient` و runtime از طریق OpenTelemetry.
- امکان افزودن exporter دلخواه OpenTelemetry؛ نمونه‌ی این راهنما از OTLP استفاده می‌کند.

## نیازمندی‌ها

- .NET 10 SDK و برنامه‌ی میزبان ASP.NET Core با Target Framework برابر `net10.0`.
- برای ارسال telemetry با OTLP، یک OpenTelemetry Collector یا backend سازگار که endpoint آن در دسترس برنامه باشد.
- برای هر وابستگی خارجی، پکیج HealthChecks متناظر و connection string آن در برنامه‌ی میزبان.

## نصب و راه‌اندازی

پس از افزودن reference این کتابخانه به پروژه‌ی سرویس، extensionهای آن را در برنامه ثبت کنید. در نمونه‌ی زیر یک health check برای PostgreSQL اضافه می‌شود و traceها و متریک‌ها از راه OTLP ارسال می‌شوند:

```csharp
using Fundation.Monitoring;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

var connectionString = builder.Configuration.GetConnectionString("MainDb")
	?? throw new InvalidOperationException("Connection string 'MainDb' is missing.");

builder.Services.AddMonitoring(
	healthChecksBuilder: checks => checks.AddNpgSql(
		connectionString,
		name: "main-database",
		tags: new[] { "database", "infra" }),
	configureOpenTelemetry: telemetry => telemetry
		.WithTracing(tracing => tracing.AddOtlpExporter())
		.WithMetrics(metrics => metrics.AddOtlpExporter()),
	serviceName: "orders-api");

var app = builder.Build();

app.UseMonitoring();

app.MapGet("/", () => Results.Ok("Orders API is running."));

app.Run();
```

در زمان اجرا آدرس Collector را تنظیم کنید. نمونه برای PowerShell:

```powershell
$env:OTEL_EXPORTER_OTLP_ENDPOINT = "http://localhost:4317"
dotnet run
```

می‌توان به‌جای پارامتر `serviceName`، متغیر محیطی `OTEL_SERVICE_NAME` را تعیین کرد. اولویت نام سرویس به‌ترتیب پارامتر، متغیر محیطی، نام assembly برنامه‌ی میزبان و در نهایت `unknown_service` است.

## APIهای مانیتورینگ

`AddMonitoring` را یک بار هنگام پیکربندی DI صدا بزنید. پارامتر `healthChecksBuilder` برای ثبت dependency checkها است. پارامتر `configureOpenTelemetry` امکان افزودن exporter یا پیکربندی بیشتر برای trace و metrics را می‌دهد. instrumentationهای ASP.NET Core، `HttpClient` و runtime به‌صورت پیش‌فرض ثبت می‌شوند.

`UseMonitoring` را یک بار در pipeline برنامه اضافه کنید. این متد Prometheus HTTP/gRPC metrics، مسیر `/metrics`، health check endpointها و HealthChecks UI را راه‌اندازی می‌کند.

### Endpointها

| مسیر | کاربرد |
| --- | --- |
| `/metrics` | متریک‌های Prometheus؛ این خروجی مستقل از exporter متریک OpenTelemetry است. |
| `/health/live` | زنده‌بودن خود برنامه؛ فقط checkهایی که tag `live` دارند بررسی می‌شوند. |
| `/health/ready` | آمادگی دریافت ترافیک؛ همه‌ی checkها به‌جز موارد tagشده با `live` بررسی می‌شوند. |
| `/healthz` | گزارش کامل همه‌ی health checkها، با قالب HealthChecks UI. |
| `/health` | گزارش JSON از checkهایی که tag `services` ندارند. |
| `/health/infra` | health checkهایی که tag `infra` دارند. |
| `/health/database` | health checkهایی که tag `database` دارند. |
| `/healthcheck-ui` | رابط کاربری HealthChecks UI. |
| `/healthcheck` | API مورد استفاده‌ی HealthChecks UI. |

برای مثال، PostgreSQL در نمونه هم tagهای `database` و `infra` دارد؛ در نتیجه در endpointهای `/health/database` و `/health/infra` نمایش داده می‌شود. برای readiness، checkهای dependency را بدون tag `live` ثبت کنید. tag `live` را فقط برای checkهایی به‌کار ببرید که زنده‌بودن process را می‌سنجند و نباید readiness را تعیین کنند. کتابخانه یک check داخلی به نام `self` با همین tag ثبت می‌کند.

## نکات اجرایی

- ثبت instrumentation به‌تنهایی telemetry را به سرویس بیرونی ارسال نمی‌کند. برای خروجی OTLP باید exporter را در `configureOpenTelemetry` اضافه کنید و `OTEL_EXPORTER_OTLP_ENDPOINT` را تنظیم کنید.
- Prometheus از مسیر `/metrics` سرو می‌شود و به تنظیم OTLP وابسته نیست؛ می‌توان از Prometheus، OTLP یا هر دو استفاده کرد.
- HealthChecks UI در این کتابخانه از storage حافظه‌ای استفاده می‌کند. داده‌های UI با restart پاک می‌شوند و بین چند replica مشترک نیستند.
- endpointهای health اطلاعات وضعیت وابستگی‌ها را برمی‌گردانند؛ در محیط production دسترسی عمومی به آن‌ها را مطابق سیاست امنیتی سرویس محدود کنید.
- از افزودن شناسه‌های پرتغییر مانند user ID یا request ID به labelهای متریک خودداری کنید تا تعداد سری‌های Prometheus بی‌رویه زیاد نشود.

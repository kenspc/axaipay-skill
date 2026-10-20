# Minimal ASP.NET Core example

A small, generic example for .NET 8 or later (minimal APIs). It shows the shape
of a correct integration: one signing helper, a payment form, both callbacks,
inquiry, and the startup log line. It is a reference, not a library. Adapt it to
the codebase you are working in: its options pattern, data access, auth and
logging conventions come first.

Everything Axaipay-specific here follows [api.md](api.md). Your own order
storage is left as an interface (`IOrderStore`) because it depends on your
database.

## Configuration

`appsettings.json` holds the non-secret values for one environment:

```json
{
  "Axaipay": {
    "MerchantId": "<merchant id>",
    "PaymentUrl": "https://staging.axaipay.my/gateway/v1/payment",
    "InquiryUrl": "https://api-staging.axaipay.my/trxn/inquiry",
    "RedirectUrl": "https://shop.example.com/pay/return",
    "BackendUrl": "https://shop.example.com/pay/callback"
  }
}
```

The two secrets come from the environment or a secret store, never from a
committed file:

```text
Axaipay__SigningKey=<signing key>
Axaipay__ApiKey=<api key>
```

Production uses `https://secured.axaipay.my/gateway/v1/payment`,
`https://api.axaipay.my/trxn/inquiry`, and the production keys, as one set.

## Axaipay.cs

```csharp
// File: Axaipay.cs
using System.Globalization;
using System.Security.Cryptography;
using System.Text;
using System.Text.Encodings.Web;
using System.Text.Json;
using Microsoft.AspNetCore.Http;

namespace Shop.Payments;

public sealed class AxaipayOptions
{
    public string MerchantId { get; set; } = "";
    public string SigningKey { get; set; } = "";
    public string ApiKey { get; set; } = "";
    public string PaymentUrl { get; set; } = "";
    public string InquiryUrl { get; set; } = "";
    public string? RedirectUrl { get; set; }
    public string? BackendUrl { get; set; }
}

public static class AxaipaySignature
{
    // Field orders are the alphabetical order of the field names (eComPay API v1.19 r1).
    public static readonly string[] PaymentFields =
        ["backendUrl", "customerEmail", "customerName", "customerPhone", "mchtId",
         "mchtTrxnId", "orderDescription", "redirectUrl", "txnAmount"];

    public static readonly string[] CallbackFields =
        ["mchtId", "mchtTrxnId", "trxnAcquirer", "trxnAmount", "trxnBankName", "trxnFpxMethod",
         "trxnFpxType", "trxnId", "trxnPaymentMethod", "trxnStatus", "trxnTime"];

    public static readonly string[] InquiryByOrderFields = ["mchtTrxnId", "merchantId"];

    // HMAC-SHA512, Base64 - not hex.
    public static string Sign(string stringToSign, string signingKey) =>
        Convert.ToBase64String(HMACSHA512.HashData(
            Encoding.UTF8.GetBytes(signingKey), Encoding.UTF8.GetBytes(stringToSign)));

    // A field that is absent contributes nothing, which is also how optional URLs drop out.
    public static string Concat(IEnumerable<string> fieldOrder, Func<string, string?> valueOf) =>
        string.Concat(fieldOrder.Select(name => valueOf(name) ?? ""));

    public static bool Matches(string expected, string? received) =>
        received is not null &&
        CryptographicOperations.FixedTimeEquals(
            Encoding.ASCII.GetBytes(expected), Encoding.ASCII.GetBytes(received));

    public static string FormatAmount(decimal amount) =>
        amount.ToString("0.00", CultureInfo.InvariantCulture);
}

public sealed record Order(
    string Id, decimal Amount, string Description,
    string CustomerName, string CustomerEmail, string CustomerPhone);

public static class AxaipayPaymentForm
{
    public static string Build(AxaipayOptions options, string mchtTrxnId, Order order)
    {
        var fields = new Dictionary<string, string>
        {
            ["mchtId"] = options.MerchantId,
            ["mchtTrxnId"] = mchtTrxnId,
            ["txnAmount"] = AxaipaySignature.FormatAmount(order.Amount),
            ["customerEmail"] = order.CustomerEmail,
            ["customerName"] = order.CustomerName,
            ["customerPhone"] = order.CustomerPhone,
            ["orderDescription"] = order.Description,
        };
        // Optional URLs are signed only when sent.
        if (!string.IsNullOrEmpty(options.RedirectUrl)) fields["redirectUrl"] = options.RedirectUrl;
        if (!string.IsNullOrEmpty(options.BackendUrl)) fields["backendUrl"] = options.BackendUrl;

        var stringToSign = AxaipaySignature.Concat(
            AxaipaySignature.PaymentFields, name => fields.GetValueOrDefault(name));
        fields["signature"] = AxaipaySignature.Sign(stringToSign, options.SigningKey);

        var html = HtmlEncoder.Default;
        var page = new StringBuilder("<!doctype html><html><body>");
        page.Append($"<form id=\"axaipay\" method=\"post\" action=\"{html.Encode(options.PaymentUrl)}\">");
        foreach (var (name, value) in fields)
            page.Append($"<input type=\"hidden\" name=\"{name}\" value=\"{html.Encode(value)}\">");
        page.Append("<button type=\"submit\">Continue to payment</button></form>");
        page.Append("<script>document.getElementById('axaipay').submit();</script>");
        page.Append("</body></html>");
        return page.ToString();
    }
}

public static class AxaipayCallback
{
    // Verify over the raw posted strings; never re-format them first.
    public static bool IsAuthentic(IFormCollection form, AxaipayOptions options)
    {
        var stringToSign = AxaipaySignature.Concat(
            AxaipaySignature.CallbackFields,
            name => form.TryGetValue(name, out var value) ? value.ToString() : null);
        var expected = AxaipaySignature.Sign(stringToSign, options.SigningKey);
        return AxaipaySignature.Matches(expected, form["signature"].ToString())
            && form["mchtId"].ToString() == options.MerchantId;
    }

    public static decimal? Amount(IFormCollection form) =>
        decimal.TryParse(form["trxnAmount"].ToString(), NumberStyles.Number,
            CultureInfo.InvariantCulture, out var amount) ? amount : null;
}

public sealed record InquiryResult(
    bool Success, string? ErrorCode, string? Message,
    int? TrxnStatus, string? TrxnId, decimal? TrxnAmount, int? TrxnPaymentMethod);

public sealed class AxaipayInquiry(HttpClient http, AxaipayOptions options)
{
    public async Task<InquiryResult> ByOrderIdAsync(string mchtTrxnId, CancellationToken ct = default)
    {
        var body = new Dictionary<string, string>
        {
            ["mchtTrxnId"] = mchtTrxnId,
            ["merchantId"] = options.MerchantId, // "merchantId" here, not "mchtId"
        };
        using var request = new HttpRequestMessage(HttpMethod.Post, options.InquiryUrl)
        {
            Content = new FormUrlEncodedContent(body),
        };
        // Credentials travel in headers, not in the body.
        request.Headers.Add("x-api-key", options.ApiKey);
        request.Headers.Add("x-signature", AxaipaySignature.Sign(
            AxaipaySignature.Concat(AxaipaySignature.InquiryByOrderFields, name => body[name]),
            options.SigningKey));

        using var response = await http.SendAsync(request, ct);
        var text = await response.Content.ReadAsStringAsync(ct);

        JsonElement root;
        try { root = JsonDocument.Parse(text).RootElement; }
        catch (JsonException)
        {
            // A 404 here usually means the payment host was configured instead of the inquiry host.
            return new InquiryResult(false, $"HTTP {(int)response.StatusCode}", null, null, null, null, null);
        }

        // Gate on root "success": "data" is present even when the inquiry failed.
        var succeeded = root.TryGetProperty("success", out var flag) && flag.ValueKind == JsonValueKind.True;
        if (!succeeded || !root.TryGetProperty("data", out var data) || data.ValueKind != JsonValueKind.Object)
            return new InquiryResult(false, Text(root, "errorCode"), Text(root, "message"), null, null, null, null);

        // The response also carries a signature over the callback fields (vendor document §3.5).
        // Our integration did not verify it, so this example does not either.
        return new InquiryResult(
            true, null, Text(root, "message"),
            ToInt(Text(data, "trxnStatus")), Text(data, "trxnId"),
            decimal.TryParse(Text(data, "trxnAmount"), NumberStyles.Number, CultureInfo.InvariantCulture, out var amount) ? amount : null,
            ToInt(Text(data, "trxnPaymentMethod")));
    }

    // Numeric fields arrive as JSON numbers; accept numeric strings too.
    private static string? Text(JsonElement element, string name) =>
        !element.TryGetProperty(name, out var value) ? null
        : value.ValueKind == JsonValueKind.String ? value.GetString()
        : value.ValueKind == JsonValueKind.Number ? value.GetRawText()
        : null;

    private static int? ToInt(string? text) =>
        int.TryParse(text, NumberStyles.Integer, CultureInfo.InvariantCulture, out var number) ? number : null;
}

public interface IOrderStore
{
    Task<Order?> FindAsync(string orderId);
    // Returns a new, unique mchtTrxnId linked to the order. Never reuse one.
    Task<string> NewAttemptAsync(string orderId);
    Task<Order?> FindByAttemptAsync(string mchtTrxnId);
    // Idempotent on trxnId, inside one database transaction: a repeated callback changes nothing.
    Task RecordResultOnceAsync(string mchtTrxnId, string trxnId, int trxnStatus);
    Task<int?> GetStatusAsync(string mchtTrxnId);
}
```

## Program.cs

```csharp
// File: Program.cs
using System.Globalization;
using Shop.Payments;

var builder = WebApplication.CreateBuilder(args);
var axaipay = builder.Configuration.GetSection("Axaipay").Get<AxaipayOptions>() ?? new AxaipayOptions();
builder.Services.AddSingleton(axaipay);
builder.Services.AddHttpClient<AxaipayInquiry>();
// builder.Services.AddScoped<IOrderStore, YourOrderStore>();

var app = builder.Build();

// Only the host selects the environment; the merchant ID can be the same on both.
// Log the derived target once. Never throw here: a payment setting must not stop the site.
static string TargetOf(string url) =>
    (Uri.TryCreate(url, UriKind.Absolute, out var uri) ? uri.Host : "") switch
    {
        "secured.axaipay.my" or "api.axaipay.my" => "Production",
        "staging.axaipay.my" or "api-staging.axaipay.my" => "Staging",
        _ => "Unknown",
    };
var paymentTarget = TargetOf(axaipay.PaymentUrl);
var inquiryTarget = TargetOf(axaipay.InquiryUrl);
if (paymentTarget == "Unknown" || paymentTarget != inquiryTarget)
    app.Logger.LogWarning("Axaipay target: payment {Payment}, inquiry {Inquiry}", paymentTarget, inquiryTarget);
else
    app.Logger.LogInformation("Axaipay target: {Target}", paymentTarget);

// Start a payment. In a real app this endpoint sits behind your auth and CSRF protection,
// and the amount comes from your order, never from the browser.
app.MapPost("/pay/start/{orderId}", async (string orderId, AxaipayOptions options, IOrderStore orders) =>
{
    var order = await orders.FindAsync(orderId);
    if (order is null) return Results.NotFound();
    var attempt = await orders.NewAttemptAsync(orderId);
    return Results.Content(AxaipayPaymentForm.Build(options, attempt, order), "text/html");
});

// Backend callback: server to server, the source of truth.
app.MapPost("/pay/callback", async (HttpRequest request, AxaipayOptions options,
    IOrderStore orders, ILogger<Program> log) =>
{
    var form = await request.ReadFormAsync();
    var attempt = form["mchtTrxnId"].ToString();
    if (!AxaipayCallback.IsAuthentic(form, options))
    {
        log.LogWarning("Axaipay callback failed verification for {Attempt}", attempt);
        return Results.Ok();
    }

    var order = await orders.FindByAttemptAsync(attempt);
    if (order is null || AxaipayCallback.Amount(form) != order.Amount)
    {
        log.LogWarning("Axaipay callback does not match an order: {Attempt}", attempt);
        return Results.Ok();
    }

    await orders.RecordResultOnceAsync(attempt, form["trxnId"].ToString(),
        int.Parse(form["trxnStatus"].ToString(), CultureInfo.InvariantCulture));
    return Results.Ok();
}).DisableAntiforgery();

// Redirect callback: the customer's browser arrives by cross-site POST, so auth cookies
// may be missing. Verify, then redirect to a GET page, where the cookies are sent again.
app.MapPost("/pay/return", async (HttpRequest request, AxaipayOptions options) =>
{
    var form = await request.ReadFormAsync();
    var attempt = AxaipayCallback.IsAuthentic(form, options) ? form["mchtTrxnId"].ToString() : "";
    return Results.Redirect($"/pay/result?attempt={Uri.EscapeDataString(attempt)}");
}).DisableAntiforgery();

// Result page. In a real app, check that the attempt belongs to the signed-in user.
app.MapGet("/pay/result", async (string attempt, IOrderStore orders, AxaipayInquiry inquiry) =>
{
    if (attempt.Length == 0) return Results.Text("We could not verify this payment.");

    // Show what the backend callback recorded; if it has not arrived yet, ask Axaipay.
    var status = await orders.GetStatusAsync(attempt);
    if (status is null)
    {
        var result = await inquiry.ByOrderIdAsync(attempt);
        status = result.Success ? result.TrxnStatus : null;
    }
    return Results.Text(status switch
    {
        11 => "Payment received.",
        null or 1 or 2 or 3 => "Payment is being processed.",
        _ => "Payment was not completed.",
    });
});

app.Run();
```

## Test against the vectors

An xUnit test that pins the signing code to the test vectors in
[api.md](api.md#test-vectors). Run it before the first call to Axaipay.

```csharp
// File: AxaipaySignatureTests.cs
using Shop.Payments;
using Xunit;

public class AxaipaySignatureTests
{
    private const string Key = "example-signing-key";

    private static string SignFields(string[] order, Dictionary<string, string> fields) =>
        AxaipaySignature.Sign(AxaipaySignature.Concat(order, name => fields.GetValueOrDefault(name)), Key);

    [Fact]
    public void Payment_without_optional_urls()
    {
        var fields = new Dictionary<string, string>
        {
            ["customerEmail"] = "buyer@example.com", ["customerName"] = "Jane Tan",
            ["customerPhone"] = "60123456789", ["mchtId"] = "DEMO_MERCHANT",
            ["mchtTrxnId"] = "ORDER-1001", ["orderDescription"] = "Pro plan",
            ["txnAmount"] = AxaipaySignature.FormatAmount(49.9m),
        };
        Assert.Equal(
            "wOiP8ZtCkJUYRdFXRo9+cRpizDyx4xghuLnIL4JnFkw+XZUolH29wVwplFhn4QhQ0O38KChrQHpq8+SshCq9rg==",
            SignFields(AxaipaySignature.PaymentFields, fields));
    }

    [Fact]
    public void Callback_fpx()
    {
        var fields = new Dictionary<string, string>
        {
            ["mchtId"] = "DEMO_MERCHANT", ["mchtTrxnId"] = "ORDER-1001", ["trxnAcquirer"] = "21",
            ["trxnAmount"] = "49.90", ["trxnBankName"] = "OCBC Bank",
            ["trxnFpxMethod"] = "B2C (Retail Banking)", ["trxnFpxType"] = "CASA",
            ["trxnId"] = "20260101120000123", ["trxnPaymentMethod"] = "20", ["trxnStatus"] = "11",
            ["trxnTime"] = "20260101120000", ["trxnBuyerName"] = "Not signed",
        };
        Assert.Equal(
            "6FvdQyQN+9S5IJX45JOBP3t8cw6UZZZ8+iuhiijxqw7GsRS0C5mH/dRJRG/i/ksExJrdD3FlAOulsUovaWK6cw==",
            SignFields(AxaipaySignature.CallbackFields, fields));
    }

    [Fact]
    public void Inquiry_by_order_id()
    {
        var fields = new Dictionary<string, string>
        {
            ["mchtTrxnId"] = "ORDER-1001", ["merchantId"] = "DEMO_MERCHANT",
        };
        Assert.Equal(
            "vVtdLnMiWLC426vnzp9JLBfE/r1NHlGAjHK8xdlKFr9zINRJU6Y+coqTNRQNvH/LFfgro/lOJSUgzjdqrScZ+Q==",
            SignFields(AxaipaySignature.InquiryByOrderFields, fields));
    }
}
```

## Notes

- The payment page uses an inline script to submit the form. If your site sends
  a Content-Security-Policy that blocks inline scripts, use a nonce or serve the
  script from a file; the button still works without it.
- `DisableAntiforgery()` matters only if the app calls `UseAntiforgery()`. The
  two callback endpoints must accept posts from Axaipay without a token.
- The callback returns 200 even when verification fails, and logs it. Whether
  Axaipay retries on other responses is not documented.

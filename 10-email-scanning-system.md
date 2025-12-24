# 10 - Hệ Thống Quét Email

> Tự động nhận và xử lý hóa đơn từ Gmail

---

## Tổng Quan

### Workflow Tự Động

```mermaid
sequenceDiagram
    participant Email as Gmail
    participant PubSub as Google Pub/Sub
    participant Webhook as Laravel Webhook
    participant Queue as Queue Job
    participant Parser as XML Parser
    participant DB as Database
    
    Email->>PubSub: Email mới đến
    PubSub->>Webhook: Push notification
    Webhook->>Queue: Dispatch GmailScanJob
    Queue->>Email: Fetch email content
    Email-->>Queue: Email + attachments
    Queue->>Parser: Parse XML invoice
    Parser->>DB: Create Purchase Invoice
    DB->>Queue: Success
    Queue-->>User: Notification
```

---

## Setup Gmail API

### Bước 1: Google Cloud Console

```
1. Truy cập: https://console.cloud.google.com
2. Tạo project mới: "QLHoaDon"
3. Enable APIs:
   - Gmail API
   - Cloud Pub/Sub API
4. Tạo OAuth 2.0 credentials
5. Tạo Service Account (cho Pub/Sub)
```

### Bước 2: Cấu Hình .env

```dotenv
GOOGLE_PROJECT_ID="your-project-id"
GOOGLE_CLIENT_ID="xxx.apps.googleusercontent.com"
GOOGLE_CLIENT_SECRET="GOCSPX-xxx"
GOOGLE_REDIRECT_URI="https://your-domain.com/auth/google/callback"
GOOGLE_PUBSUB_TOPIC="mail-checker"
```

### Bước 3: Service Account

```bash
# Download service account JSON key
# Save to: storage/app/credentials/google-service-account.json

# File structure:
{
  "type": "service_account",
  "project_id": "...",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...",
  "client_email": "...",
  ...
}
```

---

## Kết Nối Gmail

### OAuth Flow

```php
// app/Http/Controllers/Auth/GoogleController.php
public function redirect() {
    $client = $this->getGoogleClient();
    
    // Request Gmail read-only scope
    $client->setScopes([
        'https://www.googleapis.com/auth/gmail.readonly',
        'https://www.googleapis.com/auth/gmail.modify',
    ]);
    
    $authUrl = $client->createAuthUrl();
    return redirect($authUrl);
}

public function callback(Request $request) {
    $client = $this->getGoogleClient();
    $token = $client->fetchAccessTokenWithAuthCode($request->code);
    
    // Save token to company config
    $company = Auth::user()->company;
    $config = $company->config;
    
    $config->updateSetting('gmail_settings', [
        'access_token' => $token['access_token'],
        'refresh_token' => $token['refresh_token'] ?? null,
        'expires_at' => now()->addSeconds($token['expires_in']),
    ]);
    
    // Setup Pub/Sub watch
    $this->setupPubSubWatch($token['access_token']);
    
    return redirect('/settings')
        ->with('message', 'Kết nối Gmail thành công!');
}
```

---

## Pub/Sub Watch

### Đăng Ký Nhận Thông Báo

```php
// app/Services/Utils/GmailService.php
public function setupWatch(string $accessToken): array {
    $client = new \Google_Client();
    $client->setAccessToken($accessToken);
    
    $gmail = new \Google_Service_Gmail($client);
    
    $watchRequest = new \Google_Service_Gmail_WatchRequest();
    $watchRequest->setTopicName('projects/' . env('GOOGLE_PROJECT_ID') . '/topics/' . env('GOOGLE_PUBSUB_TOPIC'));
    $watchRequest->setLabelIds(['INBOX']);  // Chỉ inbox
    
    $watchResponse = $gmail->users->watch('me', $watchRequest);
    
    return [
        'historyId' => $watchResponse->getHistoryId(),
        'expiration' => $watchResponse->getExpiration(),
    ];
}
```

### Gia Hạn Watch (7 ngày/lần)

```php
// Schedule task - chạy mỗi 6 ngày
// app/Console/Kernel.php
protected function schedule(Schedule $schedule) {
    $schedule->call(function() {
        // Renew watch for all companies
        $companies = Company::whereNotNull('config->gmail_settings->access_token')->get();
        
        foreach ($companies as $company) {
            $token = $company->config->getSetting('gmail_settings.access_token');
            app(GmailService::class)->setupWatch($token);
        }
    })->weekly();
}
```

---

## Webhook Handler

### Nhận Push Notification

```php
// routes/integrations.php
Route::post('/webhook/gmail', [GmailController::class, 'webhook'])
    ->name('gmail.webhook');

// app/Http/Controllers/GmailController.php
public function webhook(Request $request) {
    // Pub/Sub sends base64 encoded message
    $message = json_decode(base64_decode($request->input('message.data')), true);
    
    $emailAddress = $message['emailAddress'];
    $historyId = $message['historyId'];
    
    // Find company by email
    $company = $this->findCompanyByEmail($emailAddress);
    
    if ($company) {
        // Dispatch job to process
        GmailScanJob::dispatch($company->id, $historyId);
    }
    
    return response()->json(['success' => true]);
}

private function findCompanyByEmail($email) {
    return Company::whereHas('users', function($q) use ($email) {
        $q->where('email', $email);
    })->first();
}
```

---

## Queue Job - Xử Lý Email

### Fetch Email Từ Gmail

```php
// app/Jobs/GmailScanJob.php
class GmailScanJob implements ShouldQueue {
    public $companyId;
    public $historyId;
    
    public function handle() {
        $company = Company::find($this->companyId);
        $gmailService = app(GmailService::class);
        
        // Get token
        $token = $company->config->getSetting('gmail_settings.access_token');
        
        // Fetch new messages since historyId
        $messages = $gmailService->getMessagesSinceHistory($token, $this->historyId);
        
        foreach ($messages as $message) {
            $this->processMessage($message, $company);
        }
    }
    
    private function processMessage($message, $company) {
        $gmailService = app(GmailService::class);
        
        // Check if has invoice attachment (XML or PDF)
        if (!$this->hasInvoiceAttachment($message)) {
            return;
        }
        
        // Download attachments
        $attachments = $gmailService->getAttachments($message->id);
        
        foreach ($attachments as $attachment) {
            if ($this->isInvoiceFile($attachment)) {
                $this->parseAndCreateInvoice($attachment, $company);
            }
        }
    }
    
    private function hasInvoiceAttachment($message) {
        $subject = $message->subject ?? '';
        $body = $message->body ?? '';
        
        // Keywords
        $keywords = ['hóa đơn', 'invoice', 'hoadon', 'bill'];
        
        foreach ($keywords as $keyword) {
            if (stripos($subject, $keyword) !== false || 
                stripos($body, $keyword) !== false) {
                return true;
            }
        }
        
        return false;
    }
    
    private function isInvoiceFile($attachment) {
        $extension = pathinfo($attachment->filename, PATHINFO_EXTENSION);
        return in_array(strtolower($extension), ['xml', 'pdf', 'zip']);
    }
}
```

---

## Parse XML Invoice

### Cấu Trúc XML Hóa Đơn Điện Tử

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Invoice>
    <InvoiceNo>0001234</InvoiceNo>
    <InvoiceDate>2025-01-24</InvoiceDate>
    <Seller>
        <Name>CÔNG TY ABC</Name>
        <TaxCode>0123456789</TaxCode>
        <Address>123 Đường ABC, Hà Nội</Address>
    </Seller>
    <Buyer>
        <Name>CÔNG TY XYZ</Name>
        <TaxCode>9876543210</TaxCode>
    </Buyer>
    <Items>
        <Item>
            <Name>Laptop Dell</Name>
            <Quantity>2</Quantity>
            <UnitPrice>15000000</UnitPrice>
            <VATRate>10</VATRate>
        </Item>
    </Items>
    <SubTotal>30000000</SubTotal>
    <VATAmount>3000000</VATAmount>
    <GrandTotal>33000000</GrandTotal>
</Invoice>
```

### Parser

```php
// app/Services/Utils/InvoiceXMLParser.php
class InvoiceXMLParser {
    public function parse(string $xmlContent): array {
        $xml = simplexml_load_string($xmlContent);
        
        return [
            'invoice_number' => (string) $xml->InvoiceNo,
            'invoice_date' => (string) $xml->InvoiceDate,
            'seller' => [
                'name' => (string) $xml->Seller->Name,
                'tax_id' => (string) $xml->Seller->TaxCode,
                'address' => (string) $xml->Seller->Address,
            ],
            'buyer' => [
                'name' => (string) $xml->Buyer->Name,
                'tax_id' => (string) $xml->Buyer->TaxCode,
            ],
            'items' => $this->parseItems($xml->Items->Item),
            'subtotal' => (float) $xml->SubTotal,
            'vat_amount' => (float) $xml->VATAmount,
            'grand_total' => (float) $xml->GrandTotal,
        ];
    }
    
    private function parseItems($items): array {
        $result = [];
        
        foreach ($items as $item) {
            $result[] = [
                'name' => (string) $item->Name,
                'quantity' => (float) $item->Quantity,
                'unit_price' => (float) $item->UnitPrice,
                'vat_rate' => (float) $item->VATRate,
            ];
        }
        
        return $result;
    }
}
```

---

## Tạo Hóa Đơn Tự Động

### Từ Dữ Liệu Đã Parse

```php
private function parseAndCreateInvoice($attachment, $company) {
    try {
        // Parse XML
        $parser = app(InvoiceXMLParser::class);
        $data = $parser->parse($attachment->content);
        
        DB::transaction(function() use ($data, $company) {
            // Find or create supplier
            $supplier = $this->findOrCreateSupplier($data['seller'], $company);
            
            // Create purchase invoice
            $invoice = PurchaseInvoice::create([
                'company_id' => $company->id,
                'supplier_id' => $supplier->id,
                'purchase_number' => $data['invoice_number'],
                'purchase_date' => $data['invoice_date'],
                'currency' => 'VND',
                'status' => 'pending',  // Chờ duyệt
                'subtotal' => $data['subtotal'],
                'vat_amount' => $data['vat_amount'],
                'grand_total' => $data['grand_total'],
                'source' => 'email',  // Đánh dấu từ email
            ]);
            
            // Create items
            foreach ($data['items'] as $itemData) {
                // Find or create product
                $product = $this->findOrCreateProduct($itemData, $company);
                
                PurchaseItem::create([
                    'purchase_id' => $invoice->id,
                    'product_id' => $product->id,
                    'quantity' => $itemData['quantity'],
                    'unit_price' => $itemData['unit_price'],
                    'vat_rate' => $itemData['vat_rate'],
                    'total_price' => $itemData['quantity'] * $itemData['unit_price'],
                ]);
            }
            
            // Notify user
            $this->notifyUser($company, $invoice);
        });
    } catch (\Exception $e) {
        // Log error
        \Log::error('Invoice parsing failed', [
            'error' => $e->getMessage(),
            'company_id' => $company->id,
        ]);
    }
}

private function findOrCreateSupplier($sellerData, $company) {
    return Supplier::firstOrCreate(
        [
            'company_id' => $company->id,
            'tax_id' => $sellerData['tax_id'],
        ],
        [
            'supplier_name' => $sellerData['name'],
            'address' => $sellerData['address'],
        ]
    );
}
```

---

## Thông Báo Người Dùng

### Real-time Notification

```php
private function notifyUser($company, $invoice) {
    // Broadcast to company channel
    event(new InvoiceReceivedViaEmail($invoice));
    
    // Send email to admins
    $admins = User::where('company_id', $company->id)
        ->whereHas('role', function($q) {
            $q->where('code', 'admin');
        })
        ->get();
    
    foreach ($admins as $admin) {
        $admin->notify(new NewInvoiceFromEmail($invoice));
    }
}
```

---

## Manual Scan

### Quét Email Thủ Công

```php
// app/Livewire/Main/Settings/GmailSettings.php
public function scanNow() {
    $company = Auth::user()->company;
    $token = $company->config->getSetting('gmail_settings.access_token');
    
    if (!$token) {
        $this->addError('gmail', 'Chưa kết nối Gmail');
        return;
    }
    
    // Dispatch job
    GmailScanJob::dispatchSync($company->id, null);
    
    $this->message = 'Đã quét email thành công!';
}
```

---

## Xử Lý Lỗi

### Refresh Token Khi Hết Hạn

```php
// app/Services/Utils/GmailService.php
private function getClient($company) {
    $client = new \Google_Client();
    
    $settings = $company->config->getSetting('gmail_settings');
    $accessToken = $settings['access_token'];
    
    $client->setAccessToken($accessToken);
    
    // Check if expired
    if ($client->isAccessTokenExpired()) {
        if (isset($settings['refresh_token'])) {
            // Refresh
            $newToken = $client->fetchAccessTokenWithRefreshToken($settings['refresh_token']);
            
            // Update
            $company->config->updateSetting('gmail_settings.access_token', $newToken['access_token']);
            
            $client->setAccessToken($newToken);
        } else {
            throw new \Exception('Token hết hạn, cần kết nối lại Gmail');
        }
    }
    
    return $client;
}
```

---

## Quick Reference

### Setup Checklist

```
✅ Google Cloud project
✅ Enable Gmail API + Pub/Sub
✅ OAuth credentials
✅ Service account JSON
✅ .env config
✅ Webhook URL public
✅ Setup watch
✅ Test with real email
```

### Flow

```
Gmail → Pub/Sub → Webhook → Queue Job → Parse XML → Create Invoice → Notify
```

---

## Tiếp Theo

✅ Email scanning đã hiểu!

**Tiếp tục:**
- [AI Chatbot](11-ai-chatbot.md)
- [Thông Báo](12-notification-system.md)
- [Cache System](13-cache-system.md)

---

<p align="center">
  <strong>Email Automation Thành Thạo! 📧</strong>
</p>

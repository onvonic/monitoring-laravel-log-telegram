# Monitoring Laravel Log with Telegram

This package provides a way to log Laravel application errors to both a file and a Telegram chat using custom helper classes.

## Installation

1. **LARAVEL TELEGRAM** [Set Telegram in your Laravel project](https://github.com/onvonic/Laravel-bot-telegram).

2. **Add the custom log channel** to your `config/logging.php`:

    ```php
    'channels' => [
        // Other channels...
        
        'log_app' => [
            'driver' => 'single',
            'path' => storage_path('logs/log_app.log'),
            'level' => 'error',
        ],
    ],
    ```

3. **Create the helper classes** in your Laravel project:

    - **TelegramLogHelper** (`app/Helpers/TelegramLogHelper.php`):

        ```php
        <?php

        namespace App\Helpers;

        use Illuminate\Support\Facades\Http;
        use Illuminate\Support\Facades\Log;

        class TelegramLogHelper
        {
            public static function sendLogMessage($data)
            {
                $botToken = config('app.telegramBotToken');
                $chatId   = '7497549486';
                $message = implode([
                    "Log Message: \n",
                    "App Name: " . config('app.name') . "\n",
                    "Method: " . $data['method'] . "\n",
                    "Module: " . $data['module'] . "\n",
                    "Status: " . $data['status'] . "\n\n",
                    "Data: " . json_encode($data['data'], JSON_PRETTY_PRINT)
                ]);
                $response = Http::post("https://api.telegram.org/bot$botToken/sendMessage", [
                    'chat_id' => $chatId,
                    'text'    => $message,
                ]);
            }
        }
        ```

    - **FileLogHelper** (`app/Helpers/FileLogHelper.php`):

        ```php
        <?php

        namespace App\Helpers;

        use Illuminate\Support\Facades\Log;

        class FileLogHelper
        {
            public static function sendLogMessage(array $data)
            {
                $status = $data['status'] ?? 'error';
                $method = $data['method'] ?? 'unknown';
                $module = $data['module'] ?? 'general';

                $logFile = storage_path("logs/logapp{$module}.log");

                // Configure the custom log channel
                config(['logging.channels.log_app.path' => $logFile]);

                // Determine the log level based on status
                $logLevel = $status === 'success' ? 'info' : 'error';

                // Log the message
                Log::channel('log_app')->$logLevel("[{$method}] " . $data['data']['error'], [
                    'file' => $data['data']['file'] ?? 'N/A',
                    'line' => $data['data']['line'] ?? 'N/A',
                ]);
            }
        }
        ```

## Usage

In your Laravel controller, you can use the helper classes to log errors to both a file and Telegram.

### Example

```php
try {
    // Your code logic here
} catch (\Exception $e) {
    $data = [
        'status' => 'error',
        'method' => 'view',
        'module' => 'users',
        'data'   => [
            'error' => $e->getMessage(),
            'file'  => $e->getFile(),
            'line'  => $e->getLine()
        ]
    ];
    
    FileLogHelper::sendLogMessage($data);
    TelegramLogHelper::sendLogMessage($data);

    return response()->json([
        'status'  => false,
        'message' => 'Failed to view data',
        'error'   => 'An internal error occurred. Check the log file.'
    ]);
}

[Тестовое задание](https://github.com/nikiforov-org/smartik/blob/main/task.md)

# Бэкенд
Используем очереди и события.

При факте доставки заказа вызываем событие OrderDelivered, которое будет содержать информацию о заказе и о клиенте. Добавляем заказ в очередь на обработку.

При добавлении заказа в очередь, вызываем метод dispatchAfterResponse у очереди, который позволит отправить ответ клиенту сразу после добавления заказа в очередь, не дожидаясь выполнения всех задач в очереди.

Создаем обработчик для данного события OrderDelivered, который будет отвечать за списание средств с клиента и печать чека. В обработчике проверяем статус списания средств через вебхук платежного агента и, если списание прошло успешно, то печатаем чек клиенту.

```
// Обработчик события
class ProcessOrderDelivered
{
    public function handle(OrderDelivered $event)
    {
        $order = $event->order;
        $client = $event->client;

        // Списываем деньги с клиента
        $paymentAgent = app(PaymentAgent::class);
        $response = $paymentAgent->charge($client->id, $order->amount);
        if ($response->isSuccessful()) {
            // Если списание прошло успешно, печатаем чек клиенту
            $printer = app(Printer::class);
            $printer->printCheck($client->id, $order->amount);
        }
    }
}

// Контроллер для факта доставки заказа
class OrderController extends Controller
{
    public function delivered(Request $request)
    {
        // Отбиваем статус на бэк
        $driver = auth()->user();
        $order = $driver->orders()->find($request->order_id);
        $order->status = 'delivered';
        $order->save();

        // Генерируем событие о доставке заказа
        event(new OrderDelivered($order, $order->client));

        return response()->json(['message' => 'Order delivered.'], 200)->dispatchAfterResponse();
    }
}

// Модель заказа
class Order extends Model
{
    public function client()
    {
        return $this->belongsTo(Client::class);
    }
}

// Событие о доставке заказа
class OrderDelivered
{
    public $order;
    public $client;

    public function __construct(Order $order, Client $client)
    {
        $this->order = $order;
        $this->client = $client;
    }
}
```
Сервисы PaymentAgent и Printer соответственно реализованы в проекте. 
Настраиваем очередь на обработку задачи OrderDelivered через драйвер Redis config/queue.php:
```
// Конфигурация очереди
return [
    'default' => env('QUEUE_CONNECTION', 'redis'),

    'connections' => [
        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
            'queue' => env('REDIS_QUEUE', 'default'),
            'retry_after' => 90,
            'block_for' => null,
        ],
    ],

    // ...
];

// Дополнительные параметры для Redis
return [
    // ...
    'redis' => [
        'cluster' => false,
        'default' => [
            'host' => env('REDIS_HOST', '127.0.0.1'),
            'password' => env('REDIS_PASSWORD', null),
            'port' => env('REDIS_PORT', 6379),
            'database' => env('REDIS_DB', 0),
        ],
    ],
];
```
Запускаем воркеры очереди для обработки задач `php artisan queue:work`.

# Фронтенд
### 1. 
Ниже приведённый код работает быстрее за счет того, что каждый раз число проверяется не со следующим, а с числом, находящимся в середине интервала, после чего интервал делится пополам и дальнейший поиск ведется только в половине интервала, в которой может быть найдено нужное число.
```
const target = 100;
let low = 1;
let high = 1000000000;
let isFind = false;

while (low <= high) {
    let mid = Math.floor((low + high) / 2);
    
    if (mid === target) {
        isFind = true;
        break;
    } else if (mid < target) {
        low = mid + 1;
    } else {
        high = mid - 1;
    }
}

if (isFind) {
    ...
}
```
Есть ещё такой способ, если мы хотим перебором, более быстрый чем в задаче, но менее оптимальный, чем код выше.
Декремент оператор быстрее инкремент оператора, поскольку не требуется выполнения операции сложения, а выполняется только операция вычитания.
Не столь эффективно, просто я об этом знаю :)

```
const target = 100;
let high = 1000000000;
let isFind = false;

for (let i = high; i--;) {
    if (i === need) isFind = true 
}

if (isFind) {
    ...
}
```
## 2. 
Проблема заключается в том, что функция isAdmin() возвращает Promise, а не булевое значение напрямую. Когда условие if (isAdmin()) выполняется, оно всегда true, даже если фактическое значение response.ok является false.

Чтобы исправить эту проблему, можно использовать await для ожидания результата выполнения функции isAdmin(), и затем проверить фактическое значение response.ok:
```
const isAdmin = async () => {
    const response = await fetch('/admin/auth')
    return response.ok
}

const auth = async () => {
    if (await isAdmin()) {
        return navigateToAdmin()
    }
    navigateToUser()
}
```

Теперь функция isAdmin() возвращает Promise, и ожидание ее результата в if (await isAdmin()) позволяет проверить фактическое значение response.ok, а не просто наличие Promise.

# SQL
Выбираем все поля из таблицы routes, где company_id == 1 и office_id == 1, и где существует запись в таблице drivers, у которой id == user_id в таблице route_users, а также route_id == id в таблице routes, и где user_type равен строке 'App\\Models\\Driver', и где name, surname или middlename содержат подстроку 'гвоз'. Результаты сортируются по убыванию id и ограничиваются до 15 записей.

Очевидно, это поиск маршрутов, связанных с водителями, у которых есть буквы "гвоз" в ФИО, а также связанные с определенной компанией и офисом.

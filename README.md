# Americor's PHP Developer Test Assignment review
This review will be divided into blocks, each focused on specific types of issues. The first block will address minor database-related considerations. Following that, I will delve into PHP code style and minor adjustments that can enhance the codebase. Subsequently, I'll outline major code improvements feasible at the current project stage. Finally, I'll discuss future enhancements, keeping in mind the project's anticipated growth.
## DB-related considerations (minor)
* Inconsistencies in timestamp fields are observed across the database schema. While some tables feature 'ins_ts' fields to track timestamps (such as 'call', 'fax', and 'history' tables), others, like 'sms' and 'task', lack them entirely. Moreover, the 'users' table uses the 'created_at' and 'updated_at' fields for the same (or almost the same) purposes. These fields not only have different names but also have different data types. 
* Several tables feature 'status' and 'direction' fields containing numeric values, yet their specific meanings are not transparent at the database level without referencing the PHP code. To enhance readability and context, we can add dictionary tables or at least add comments to the fields.
```sql
alter table fax
    modify direction smallint default 0 not null comment '0 for incoming, 1 for outgoing';
```
* Using a dictionary for these fields would not only offer clarity but also let us be sure that we insert existing values. There are some inconsistencies in data types across tables, for example 'call.direction' being stored as 'smallint' while 'sms.direction' as 'tinyint(3)'. Using the same data types simplifies maintenance and can improve database performance.
* The order of fields in certain tables, such as the 'columns' table, seems pretty random. For example, I expect to see the 'ins_ts' field at the end of the field list, with 'user_id' and 'customer_id' immediately after the 'id' field. Reorganizing the field order in this manner would enhance readability and maintain consistency in the database schema.
* The 'user' table contains entries with various email states, including users with valid email addresses, users with empty email fields, and users with email fields set to null. This inconsistency may lead to potential issues related to data integrity and application functionality.

## Code style and small adjustments
* I propose to explicitly indicate variable types and return types for methods throughout the codebase. This practice not only enhances code clarity but also helps in  identifying potential issues early in the development process. Implementing this standard across the entire project can boost its  maintainability.
* I am not against using traits, like some authors. You can read this article for better understanding this position: [When to use a trait?](https://matthiasnoback.nl/2022/07/when-to-use-a-trait/)
In my understanding it is ok if this allows you to decrease the complexity of the code. That’s why the  idea of using one very specific trait as we do with ObjectNameTrait seems a bit flawed to me. This trait doesn’t help us to avoid code duplication, and just make the code less transparent. But it is just a choice, I can’t call it a mistake, but still want to mention.
* Using constants instead of magic numbers.
```php
public $duration = 720;
```

For example instead of just using number 720, we can create 
```php
const TWELVE_MINUTES = 720;
```
and use `TWELVE_MINUTES` instead of `720`
* Using `==` instead of `===` sometimes may cause issues.
```php
if (
    $this->status == self::STATUS_NO_ANSWERED
    && $this->direction == self::DIRECTION_INCOMING
) {
    return Yii::t('app', 'Missed Call');
}
``` 
Here both constants are equal to 0, and if $this->status is null, this will return true. Maybe this exact code will still work, but the situation when `$this->status` becomes null and we don’t expect this can send us a signal that something is wrong.
* Using ?? operator. Code like this:
```php
return isset($detail->changedAttributes->{$attribute}) ? $detail->changedAttributes->{$attribute} : null;
```
Can be safely replaced like this: 
```php
return $detail->changedAttributes->{$attribute} ?? null;
```
It looks better and there is less possibility of typo.

* Even if some status is not being used, this constant called STATUS_SUCCESS is equal to status_delivered, and this may cause confusion. I propose to delete constants we don’t use anymore.
```php
    const STATUS_DELIVERED = 13;
    const STATUS_SUCCESS = 13;
```
## Major code improvements
* this code smells:
  
![smell](https://github.com/svetlakoviv/review/assets/59096489/9b3b7956-c249-40de-9eda-dffc59895c29)

Adding the new object types will make it even more unreadable. I propose to use separate classes for each object type and move the logic to these classes.
1. we can create an interface
```php
namespace app\widgets\HistoryList\Objects;

use app\models\History;

interface ListObjectInterface
{
    public static function getBody(History $model);
}
```

2. then classes for each object implementing this interface
```php
namespace app\widgets\HistoryList\Objects;

use app\models\Customer;
use app\models\History;
use Yii;

class EventListObject implements ListObjectInterface
{

    public static function getBody(History $model)
    {
        return match ($model->event) {
            History::EVENT_CUSTOMER_CHANGE_TYPE => "$model->eventText " .
                (Customer::getTypeTextByType($model->getDetailOldValue('type')) ?? "not set") . ' to ' .
                (Customer::getTypeTextByType($model->getDetailNewValue('type')) ?? "not set"),
            History::EVENT_CUSTOMER_CHANGE_QUALITY => "$model->eventText " .
                (Customer::getQualityTextByQuality($model->getDetailOldValue('quality')) ?? "not set") . ' to ' .
                (Customer::getQualityTextByQuality($model->getDetailNewValue('quality')) ?? "not set"),
            default => $model->eventText,
        };

    }
}
```
3. Now the history list helper class will look like this:

![no smell](https://github.com/svetlakoviv/review/assets/59096489/69b44b13-e6c0-4592-8a1a-9b2c1c58e934)


New classes we created for each object type. If the object list class isn't implemented yet we will use the default list object to avoid issues.
Now if we need to handle a new list object we simply add the class and add this class into the `OBJECTS` array.
<img width="298" alt="image" src="https://github.com/svetlakoviv/review/assets/59096489/a30befc6-9bf7-4deb-89d2-71a052906c4f">


* Adding Dependency injection.
  Instead creating objects like HistorySearch using new
```php
 public function actionExport($exportType)
    {
        $model = new HistorySearch();

        return $this->render('export', [
            'dataProvider' => $model->search(Yii::$app->request->queryParams),
            'exportType' => $exportType,
            'model' => $model
        ]);
    }
```

We can inject these objects right into the controller’s actions: 

```php
 public function actionExport($exportType, HistorySearch $model)
    {
        return $this->render('export', [
            'dataProvider' => $model->search(Yii::$app->request->queryParams),
            'exportType' => $exportType,
            'model' => $model
        ]);
    }
```

All we need is just to register the class.
```php
\Yii::$container->set('app\models\search\HistorySearch');
```
The same idea with History List class, we can inject a HistorySearch object during __construct and then use it in method run().
```php
class HistoryList extends Widget
{
    protected HistorySearch $historySearch;

    function __construct(HistorySearch $historySearch, $config = [])
    {
        $this->historySearch = $historySearch;

        parent::__construct($config);
    }

    /**
     * @return string
     */
    public function run()
    {
        return $this->render('main', [
            'model' => $this->historySearch,
            'linkExport' => $this->getLinkExport(),
            'dataProvider' => $this->historySearch->search(Yii::$app->request->queryParams)
        ]);
    }

```

Why is this important? This works the same way as before but now the code becomes more testable and decoupled. Now if we decide to create a unit test for this class using a mock model instead of the real one we can easily do it. Same with logging and other common things we don’t want to see during unit tests.

* Separation of logic and presentation.
Many views of this project mix logic and presentation. For example look at the file _item.php, we can see that the code is quite complex and has logic inside. The example of mixing is file widgets/HistoryList/views/_item.php:

![smell2](https://github.com/svetlakoviv/review/assets/59096489/730f35fe-2262-43b0-b5a7-c2675786bb55)

Instead of just rendering something like ‘footer’ => $footer, this code is trying to understand if this is an incoming or outgoing message. This can make even small changes really challenging and tedious.
Also adding new object types will cause this file grow bigger and become even more difficult to understand. Let’s change it.

1. Go to widgets/HistoryList/views/main.php, change using of _item.php:
```php
'itemView' => '_item',
```
to calling the renderObject method from HistoryList widget
```php
'itemView' => [$this->context, 'renderObject'],
```
2. this method is simple:
```php
public function renderObject($model)
{
    return HistoryListHelper::render($model, $this);
}
```
3. Now the interface will be changed, we add 1 more method to this:
```
interface ListObjectInterface
{
    public static function getBody(History $model);

    public static function render(History $model, object $context = null);

}
```

4. add render method to each list object class like this (this is for CallListObject):

```php
public static function render(History $model, object $context = null)
{
    $call = $model->call;
    $answered = $call && $call->status == Call::STATUS_ANSWERED;
    $footer = isset($call->applicant) ? "Called <span>{$call->applicant->name}</span>" : null;
    $iconIncome = $answered && $call->direction == Call::DIRECTION_INCOMING;
    $body = self::getBody($model);
    $comment = $call->comment ?? '';


    return Yii::$app->getView()->render('_item_common', [
        'user' => $model->user,
        'content' => $comment,
        'body' => $body,
        'footerDatetime' => $model->ins_ts,
        'footer' => $footer,
        'iconClass' => $answered ? 'md-phone bg-green' : 'md-phone-missed bg-red',
        'iconIncome' => $iconIncome
    ], $context);

}
```

5. now HistroryListHelper will look like this:
```php
class HistoryListHelper
{
    const OBJECTS = [
        'call' => 'CallListObject',
        'sms' => 'SmsListObject',
        'fax' => 'FaxListObject',
        'task' => 'TaskListObject',
        'event' => 'EventListObject',
    ];

    const NAMESPACE = 'app\\widgets\\HistoryList\\Objects\\';

    public static function getBodyByModel(History $model): string
    {
        $class = self::OBJECTS[$model->object] ?? 'DefaultListObject';
        $class = self::NAMESPACE.$class;

        return $class::getBody($model);
    }

    public static function render(History $model, object $context = null): string
    {
        $class = self::OBJECTS[$model->object] ?? 'DefaultListObject';
        $class = self::NAMESPACE.$class;

        return $class::render($model, $context);
    }
}
```

Now after we do this for all types of objects, our page will look exactly as before, but now it will be super easy to add more object types to our system. 
All we need to do is to add a separate class that will implement the ListObjectInterface.
This option can be improved further, by adding parent classes for various cases. My goal here is just to show the direction in which we can move. 

* We need to use normal routing. Instead of having urls like index.php?foo=bar we should use routing like is written here: https://www.yiiframework.com/doc/guide/2.0/en/runtime-routing

## Possible future enhancements
I already showed how to add more object types, so this will not cause us a big problem, but let’s now speak about the file we export. This file will grow. Currently we just get all the data from table history and create a CSV file from this. So we generate the full file from scratch each time, despite the fact that most of the information will be the same from the last time.
If this table is only used for storing history, then the older rows will stay the same, and the part of new rows added recently will shrink. The easiest solution is just to save the previous CSV file, check the latest row in this file, and start our export from the newest row. If there are no newest rows, then we return the old file. If there are some, we just create a new CSV, glue both files together, and return to the user, saving the new file. This way will be good enough and easy to maintain. If we need to regenerate the file for some reason, we can just delete the old one and the export will start from the very beginning.
We can think about more tricky approaches that will allow us to change the old data. For example, we can have more than one CSV file; we can split it by size. Let’s imagine we have n parts of a CSV file, which we can glue and return to the user. And we need to make some changes in one of the old rows in the DB. Then we can just regenerate the part we need using the fact that for each part we know the first and last ID.
Of course, we need to think about some things like multiple users starting CSV export at once. If the export is already going then all new requests shouldn’t start the new export, but get the result of the one that is in process. And so on. There are many adjustments we can do, but this is a test case and not a real system, so I don’t want to dig too deep.

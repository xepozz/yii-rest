控制器
===========

在创建资源类和指定资源格输出式化后，下一步就是创建控制器操作将资源通过RESTful APIs展现给终端用户。

Yii 提供两个控制器基类来简化创建RESTful 操作的工作:[[Yiisoft\Yii\Rest\Controller]] 和 [[Yiisoft\Yii\Rest\ActiveController]]，
两个类的差别是后者提供一系列将资源处理成[Active Record](db-active-record.md)的操作。
因此如果使用[Active Record](db-active-record.md)内置的操作会比较方便，可考虑将控制器类
继承[[Yiisoft\Yii\Rest\ActiveController]]，它会让你用最少的代码完成强大的RESTful APIs.

[[Yiisoft\Yii\Rest\Controller]] 和 [[Yiisoft\Yii\Rest\ActiveController]] 提供以下功能，一些功能在后续章节详细描述：

* HTTP 方法验证;
* [内容协商和数据格式化](rest-response-formatting.md);
* [认证](rest-authentication.md);
* [频率限制](rest-rate-limiting.md).

[[Yiisoft\Yii\Rest\ActiveController]] 额外提供一下功能:

* 一系列常用操作: `index`, `view`, `create`, `update`, `delete`, `options`;
* 对操作和资源进行用户认证.


## 创建控制器类 <span id="creating-controller"></span>

当创建一个新的控制器类，控制器类的命名最好使用资源名称的单数格式，例如，提供用户信息的控制器
可命名为`UserController`.

创建新的操作和Web应用中创建操作类似，唯一的差别是Web应用中调用`render()`方法渲染一个视图作为返回值，
对于RESTful操作直接返回数据，[[Yiisoft\Yii\Rest\Controller::serializer|serializer]] 和
[[yii\web\Response|response object]] 会处理原始数据到请求格式的转换，例如

```php
public function actionView($id)
{
    return User::findOne($id);
}
```


## 过滤器 <span id="filters"></span>

[[Yiisoft\Yii\Rest\Controller]]提供的大多数RESTful API功能通过[过滤器](structure-filters.md)实现.
特别是以下过滤器会按顺序执行：

* [[yii\filters\ContentNegotiator|contentNegotiator]]: 支持内容协商，在 [响应格式化](rest-response-formatting.md) 一节描述;
* [[yii\filters\VerbFilter|verbFilter]]: 支持HTTP 方法验证;
  the [Authentication](rest-authentication.md) section;
* [[yii\filters\auth\AuthMethod|authenticator]]: 支持用户认证，在[认证](rest-authentication.md)一节描述;
* [[yii\filters\RateLimiter|rateLimiter]]: 支持频率限制，在[频率限制](rest-rate-limiting.md) 一节描述.

这些过滤器都在[[Yiisoft\Yii\Rest\Controller::behaviors()|behaviors()]]方法中声明，
可覆盖该方法来配置单独的过滤器，禁用某个或增加你自定义的过滤器。
例如，如果你只想用HTTP 基础认证，可编写如下代码：

```php
use yii\filters\auth\HttpBasicAuth;

public function behaviors()
{
    $behaviors = parent::behaviors();
    $behaviors['authenticator'] = [
        'class' => HttpBasicAuth::class,
    ];
    return $behaviors;
}
```


## 继承 `ActiveController` <span id="extending-active-controller"></span>

如果你的控制器继承[[Yiisoft\Yii\Rest\ActiveController]]，应设置[[Yiisoft\Yii\Rest\ActiveController::modelClass|modelClass]] 属性
为通过该控制器返回给用户的资源类名，该类必须继承[[Yiisoft\Db\ActiveRecord]].


### 自定义操作 <span id="customizing-actions"></span>

[[Yiisoft\Yii\Rest\ActiveController]] 默认提供一下操作:

* [[Yiisoft\Yii\Rest\IndexAction|index]]: 按页列出资源;
* [[Yiisoft\Yii\Rest\ViewAction|view]]: 返回指定资源的详情;
* [[Yiisoft\Yii\Rest\CreateAction|create]]: 创建新的资源;
* [[Yiisoft\Yii\Rest\UpdateAction|update]]: 更新一个存在的资源;
* [[Yiisoft\Yii\Rest\DeleteAction|delete]]: 删除指定的资源;
* [[Yiisoft\Yii\Rest\OptionsAction|options]]: 返回支持的HTTP方法.

所有这些操作通过[[Yiisoft\Yii\Rest\ActiveController::actions()|actions()]] 方法申明，可覆盖`actions()`方法配置或禁用这些操作，
如下所示：

```php
public function actions()
{
    $actions = parent::actions();

    // 禁用"delete" 和 "create" 操作
    unset($actions['delete'], $actions['create']);

    // 使用"prepareDataProvider()"方法自定义数据provider 
    $actions['index']['prepareDataProvider'] = [$this, 'prepareDataProvider'];

    return $actions;
}

public function prepareDataProvider()
{
    // 为"index"操作准备和返回数据provider
}
```

请参考独立操作类的参考文档学习哪些配置项有用。


### 执行访问检查 <span id="performing-access-check"></span>

通过RESTful APIs显示数据时，经常需要检查当前用户是否有权限访问和操作所请求的资源，
在[[Yiisoft\Yii\Rest\ActiveController]]中，可覆盖[[Yiisoft\Yii\Rest\ActiveController::checkAccess()|checkAccess()]]方法来完成权限检查。

```php
/**
 * Checks the privilege of the current user. 检查当前用户的权限
 *
 * This method should be overridden to check whether the current user has the privilege
 * to run the specified action against the specified data model.
 * If the user does not have access, a [[ForbiddenHttpException]] should be thrown.
 * 本方法应被覆盖来检查当前用户是否有权限执行指定的操作访问指定的数据模型
 * 如果用户没有权限，应抛出一个[[ForbiddenHttpException]]异常
 *
 * @param string $action the ID of the action to be executed
 * @param \yii\base\Model $model the model to be accessed. If null, it means no specific model is being accessed.
 * @param array $params additional parameters
 * @throws ForbiddenHttpException if the user does not have access
 */
public function checkAccess($action, $model = null, $params = [])
{
    // 检查用户能否访问 $action 和 $model
    // 访问被拒绝应抛出ForbiddenHttpException 
    if ($action === 'update' || $action === 'delete') {
        if ($model->author_id !== \Yii::$app->user->id)
            throw new \yii\web\ForbiddenHttpException(sprintf('You can only %s articles that you\'ve created.', $action));
    }
}
```

`checkAccess()` 方法默认会被[[Yiisoft\Yii\Rest\ActiveController]]默认操作所调用，如果创建新的操作并想执行权限检查，
应在新的操作中明确调用该方法。

> Tip: 可使用[Role-Based Access Control (RBAC) 基于角色权限控制组件](security-authorization.md)实现`checkAccess()`。

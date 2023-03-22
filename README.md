# Паттерн "Статический мост"

## Структура

1. Сначала в документе описаны задачи
2. Далее - решения, имеющие те или иные недостатки
3. Под конец описано решение через паттерн "Статический мост", решающие эти недостатки

## Исходная позиция

Есть класс-сервис ProductsManager, содержащий в себе всю логику работу с товарами.

```php
class ProductsManager {
	/**
	 * Возвращает данные о товаре.
	 */
	public function get (int $id): Product {
		return new Product($id, 'Кресло');
	}
}



class Product {
	private int    $_id;
	private string $_name;
	private string $_photoUrl = '';

	public function __construct ($id, $name) {
		$this->_id   = $id;
		$this->_name = $name;
	}

	public function getId (): int { return $this->_id; }

	public function getName (): string { return $this->_name; }

	public function getPhotoUrl (): string { /* ... */ }
}
```

Сразу он выбирает из БД только ID и имя товара.

```php
$product = $dependencyInjectionContainer->productsManager->get(1);
var_dump($product->getId(), $product->getName());
```



## Задача 1: получение фото товара

Для того, чтобы получить фото товара, нужно обратиться к какому-то относительно тяжёлому сервису, да и фото нужно далеко не всегда. Поэтому в объекте `Product` поле `$_photoUrl` изначально не инициализировано.

Тем не менее, получение фотки должно происходить прозрачно, как и обращение к ID или имени. Так как вся логика работы инкапсулирована внутри `ProductsManager`, объект `Product` для подгрузки фото налету должен обратиться к соответствующему методу `ProductsManager`.

Для этого классы должны быть написаны следующим образом:

1. `ProductsManager` в качестве зависимости передаёт в инициализируемый объект самого себя.
2. `ProductsManager` реализует метод `getPhotoUrl()` для получения картинки.
3. `Product` тоже реализует метод `getPhotoUrl()`, обращающийся к `getPhotoUrl()` сервиса.

```php
class ProductsManager {
	public function get (int $id): Product {
		return new Product($id, 'Вася', $this);
	}

	public function getPhotoUrl (int $productId): string {
		return $this->_heavyPhotosService->get($productId)->url;
	}
}



class Product {
	private ProductsManager $_productsManager;

	private int    $_id;
	private string $_name;
	private string $_photoUrl = '';

	public function __construct (int $id, string $name, ProductsManager $productsManager) {
		$this->_id   = $id;
		$this->_name = $name;
		$this->_productsManager = $productsManager;
	}

	public function getId (): int { return $this->_id; }

	public function getName (): string { return $this->_name; }

	public function getPhotoUrl (): string {
		return $this->_productsManager->getPhotoUrl($this->_id);
	}
}
```



## Задача 2: сериализация и десериализация объекта Product

Для ускорения быстродействия сайта все полученные товары могут кэшироваться через сериализацию. При сериализации фото товара может быть уже получено из базы, или нет. Соответственно, когда товар получен из кэша, обращение к его методу `getPhotoUrl()` должно налету подгрузить картинку, обращением к сервису.

### Возникающие проблемы

1. При сериализации в кэш будет попадать и тяжёлый объект сервиса `UsersManager`, со всеми его зависимостями.
2. При десериализации у этого сервиса не будут восстанавливаться подключения к ресурсам - соответственно, методы `getPhotoUrl()` просто не будут работать.



## Решение 1: `__sleep()` плюс `__wakeup()`

Добавляем метод `__sleep()`, указывающий только те поля, которые нужно сохранять при сериализации. И метод `__wakeup()`, восстанавливающий подключение к сервису  `ProductsManager`.

```php
class Product {
	public function __sleep (): array {
		return ['_id', '_name', '_photoUrl'];
	}

	public function __wakeup (): array {
		$this->_productsManager = $GLOBALS['dependencyInjectionContainer']->productsManager;
	}
}
```

### Проблемы

1. Нужно всегда следить за списком сериализуемых полей в `__sleep()` - добавляя или удаляя ключи синхронно с изменением полей.
2. Код получения сервиса из контейнера зависимостей вбит жёстко, что нарушает принцип использования инъекции зависимостей.
3. Невозможно нормальное внедрение мок-объектов для тестирования



## Решение 2: дополнительный код для сериализации/десериализации

```php
class ProductsManager {
	public function deserializeProductsList (array $productsCacheString): array {
		$productsList = unserialize($productsCacheString);
		foreach ($productsList as $product) {
			$product->setManager($this);
		}
	}
}



class Product {
	public function setManager (ProductsManager $productsManager) {
		$this->_productsManager = $productsManager;
	}

	public function __sleep (): array {
		return ['_id', '_name', '_photoUrl'];
	}
}
```

### Проблемы

1. Та же проблема с необходимостью следить за актуальностью полей в `__sleep()`
2. Простая десериализация через `unserialize()` не работает - нужно использовать вспомогательный метод `ProductsManager::deserializeProductsList()`. Если в кэше содержится много разновидностей данных, то нужно знать, какие десериализуются через `unserialize()`, а какие - через специальные методы.



## Решение: паттерн "Статический мост"

Переписываем код следующим образом:

1. Дополнительно к сервисам-объектам, лежащим в контейнере зависимостей, создаём сопутствующие классы-сервисы, доступные глобально. Или один класс-сервис с необходимым количеством методов.
2. При инициализации сервиса одним из параметров указываем, какое имя класса он должен прокидывать в создаваемые объекты для использования в роли статического моста.
3. Объект `Product` обращается к сервису через промежуточный глобально доступный статический класс, который был указан в параметрах при инициализации объекта.

```php
class ProductsManager {
	private $_staticBridgeClassName = '';

	public function __construct (array $params) {
		//...
		//Сохранение настроек и зависимостей
		//...
		$this->_staticBridgeClassName = $params['staticBridgeClassName'];
	}

	public function get (int $id): Product {
		return new Product($id, 'Вася', $this->_staticBridgeClassName);
	}

	public function getPhotoUrl (int $productId): string {
		return $this->_heavyPhotosService->get($productId)->url;
	}
}



$dependencyInjectionContainer->productsManager = new ProductsManager(
	$params + [
		'staticBridgeClassName' => '\Our\Namespace\ProductsManagerStaticBridge',
	]
);

class ProductsManagerStaticBridge {
	public function getManager (): ProductsManager {
		return $GLOBALS['dependencyInjectionContainer']->productsManager();
	}
}



class Product {
	private string $_staticBridgeClassName;

	private int    $_id;
	private string $_name;
	private string $_photoUrl = '';

	public function __construct (int $id, string $name, string $staticBridgeClassName) {
		$this->_id   = $id;
		$this->_name = $name;
		$this->_staticBridgeClassName = $staticBridgeClassName;
	}

	public function getId (): int { return $this->_id; }

	public function getName (): string { return $this->_name; }

	public function getPhotoUrl (): string {
		return ($this->_staticBridgeClassName)::getManager()->getPhotoUrl($this->_id);
	}
}
```

### Преимущества

1. В кэш сохраняются только нужные поля
2. Нет необходимости мудрить с методами `__sleep()` и `__wakeup()`.
3. Любые объекты десериализуются обычным `deserialize()`.
4. Отсутствует жёстко прописанный код подключения к сервису.
5. Для юнит-тестов в объекты можно прокидывать тестовые сервисы, указывая другой класс моста.
6. Работает со вложенными объектами.

### Недостатки

1. При какой-либо смене системы мостов нужно будет сбрасывать соответствующие кэши.
2. Обращение к глобальному классу - как вынужденная мера для реализации этого простого паттерна.

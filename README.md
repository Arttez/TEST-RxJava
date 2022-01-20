RxJava используется реактивная модель, не совсем понятно, как тестировать Observable с помощью различных методов assert. Для этого есть два основных способа. Первый способ использование метода toBlocking для Observable, который позволяет получить конкретные элементы из Observable явным образом:

@Test
public void testObservable() throws Exception {
   Observable<Integer> observable = Observable.just(1);
   int result = observable.toBlocking().first();
   assertEquals(1, result);
}
1
2
3
4
5
6
@Test
public void testObservable() throws Exception {
   Observable<Integer> observable = Observable.just(1);
   int result = observable.toBlocking().first();
   assertEquals(1, result);
}
 

Нужно также сказать, что метод toBlocking многие используют неправильно. Некоторые разработчики считают, что с помощью метода toBlocking можно использовать RxJava в качестве замены Stream API из Java 8. Это неправильный подход, нарушающий саму суть реактивного программирования.

Есть и более продвинутый способ, как тестировать Observable. Для этого существует объект TestSubscriber, с помощью которого можно подписаться на Observable, а потом проверить, какие данные поступили в этот подписчик, какие произошли ошибки и другое. Воспользоваться объектом TestSubscriber для проверки данных в Observable можно, например, так:

@Test
public void testObservableWithSubscriber() throws Exception {
   Observable<Integer> observable = Observable.just(3, 4, 5);
   TestSubscriber<Integer> testSubscriber = new TestSubscriber<>();
   observable.subscribe(testSubscriber);
  
   testSubscriber.assertValues(3, 4, 5);
   testSubscriber.assertCompleted();
}
1
2
3
4
5
6
7
8
9
@Test
public void testObservableWithSubscriber() throws Exception {
   Observable<Integer> observable = Observable.just(3, 4, 5);
   TestSubscriber<Integer> testSubscriber = new TestSubscriber<>();
   observable.subscribe(testSubscriber);
  
   testSubscriber.assertValues(3, 4, 5);
   testSubscriber.assertCompleted();
}
 

Похожим образом можно проверить и возникшие ошибки:

@Test
public void testObservableError() throws Exception {
   Observable<Object> observable = Observable.concat(
           Observable.just(1, 2),
           Observable.error(new Exception())
   );
   TestSubscriber<Object> testSubscriber = new TestSubscriber<>();
   observable.subscribe(testSubscriber);

   testSubscriber.assertValues(1, 2);
   testSubscriber.assertError(Exception.class);
}
1
2
3
4
5
6
7
8
9
10
11
12
@Test
public void testObservableError() throws Exception {
   Observable<Object> observable = Observable.concat(
           Observable.just(1, 2),
           Observable.error(new Exception())
   );
   TestSubscriber<Object> testSubscriber = new TestSubscriber<>();
   observable.subscribe(testSubscriber);
 
   testSubscriber.assertValues(1, 2);
   testSubscriber.assertError(Exception.class);
}
 

Есть также проблема при использовании асинхронности RxJava. Допустим, у нас есть Observable, который асинхронно возвращает данные (например, результат запроса с сервера). Напишем такой тест:

@Test
public void testAsyncOperation() throws Exception {
   Observable<Integer> observable = Observable.just(1)
           .delay(1, TimeUnit.SECONDS, Schedulers.io())
           .observeOn(Schedulers.immediate());

   TestSubscriber<Integer> testSubscriber = new TestSubscriber<>();
   observable.subscribe(testSubscriber);

   testSubscriber.assertValues(1);
}
1
2
3
4
5
6
7
8
9
10
11
@Test
public void testAsyncOperation() throws Exception {
   Observable<Integer> observable = Observable.just(1)
           .delay(1, TimeUnit.SECONDS, Schedulers.io())
           .observeOn(Schedulers.immediate());
 
   TestSubscriber<Integer> testSubscriber = new TestSubscriber<>();
   observable.subscribe(testSubscriber);
 
   testSubscriber.assertValues(1);
}
 

В этом тесте задается задержка перед возвращением элемента, при этом задержка выполняется в io Scheduler. Если запустить его, то тест завершится неудачно, хотя казалось, что все должно отработать корректно, ведь мы передаем какие-то данные, а потом подписываемся на них. Но JUnit тестирует все последовательно, и поэтому он не может проверить работу асинхронных программ (так как тест завершится раньше, чем будет завершен какой-то асинхронный метод). Такая ситуация может встречаться часто, если в Presenter-е вы будете выполнять какую-то логику, требующую работу в фоне. Можно пытаться выставить специальные задержки, но RxJava предоставляет способ проще, а именно дает нам возможность переопределить методы Schedulers.io и другие специальным образом. Для этого служит класс RxJavaHooks, который можно использовать следующим образом:

RxJavaHooks.setOnIOScheduler(scheduler -> Schedulers.immediate());
1
RxJavaHooks.setOnIOScheduler(scheduler -> Schedulers.immediate());
 

Теперь, когда мы будем использовать Schedulers.io, нам вернется Schedulers.immediate, и тогда тест будет выполнять в текущем потоке. Для тестов почти всегда нужно переопределять Schedulers таким образом.

AndroidSchedulers.mainThread в тестах на JUnit также не сработает, так как он требует использования класса Looper из системы Android. Поэтому и его нужно заменить схожим образом:

RxAndroidPlugins.getInstance().registerSchedulersHook(new RxAndroidSchedulersHook() {
   @Override
   public Scheduler getMainThreadScheduler() {
       return Schedulers.immediate();
   }
});
1
2
3
4
5
6
RxAndroidPlugins.getInstance().registerSchedulersHook(new RxAndroidSchedulersHook() {
   @Override
   public Scheduler getMainThreadScheduler() {
       return Schedulers.immediate();
   }
});
 

Чтобы не писать такое переопределение Schedulers в каждом тесте, их можно либо вынести в static метод, который можно будет вызывать в методе, аннотированном как Before, либо создать свой Runner, либо использовать TestRule. Пример подмены Schedulers с помощью TestRule можно найти в ссылках в конце лекции.

Измерение покрытия кода тестами
Также очень важно знать и уметь измерять, насколько полно и качественно вы тестируете свой код. Для этого существуют инструменты проверки процента покрытия кода тестами. Во-первых, это стандартный запуск тестов из Android Studio с измерения покрытия. Такой способ позволяет посмотреть, какой код был выполнен в ходе тестов, а какой остался нетронутым. Это позволяет очень удобно найти участки кода, которые не были протестированы, и написать тесты для них.

Во-вторых, есть более совершенные инструменты, такие как JaCoCo (Java Code Coverage), которые позволяют проверять процент покрытия кода тестами более детально. JaCoCo позволяет отслеживать покрытие различных веток развития программы, а не только обычное выполнение кода по строками.

Например, у нас есть следующий метод:

public int someMethod(boolean first, boolean second) {
   if (first || second) {
       return 1;
   }
   return 2;
}
1
2
3
4
5
6
public int someMethod(boolean first, boolean second) {
   if (first || second) {
       return 1;
   }
   return 2;
}
 

И тесты для него:

@Test
public void testSomeMethodCondition() throws Exception {
   int result = someMethod(false, true);
   assertEquals(1, result);
}

@Test
public void testSomeMethodDefault() throws Exception {
   int result = someMethod(false, false);
   assertEquals(2, result);
}
1
2
3
4
5
6
7
8
9
10
11
@Test
public void testSomeMethodCondition() throws Exception {
   int result = someMethod(false, true);
   assertEquals(1, result);
}
 
@Test
public void testSomeMethodDefault() throws Exception {
   int result = someMethod(false, false);
   assertEquals(2, result);
}
 

Для таких тестов при измерении покрытия из Android Studio вы получите результат в 100% (так как в ходе тестов были выполнены хотя бы раз все строчки метода), а при измерении покрытия с помощью JaCoCo вы получите результат, что есть еще случаи, которые вы не протестировали (различные варианты использования булевских переменных). К сожалению, JaCoCo не указывает, какой именно участок вашего кода не протестирован, так как он работает на уровне байт-кода и не знает о вашем исходном коде. Но JaCoCo все равно является мощным инструментом, который нужно использовать в своих проектах.

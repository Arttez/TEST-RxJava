# TEST-RxJava
использование метода toBlocking для Observable, который позволяет получить конкретные элементы из Observable

@Test
public void testObservable() throws Exception {
   Observable<Integer> observable = Observable.just(1);
   int result = observable.toBlocking().first();
   assertEquals(1, result);
}

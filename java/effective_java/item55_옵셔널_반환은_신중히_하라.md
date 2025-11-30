# item 55. 옵셔널 반환은 신중히 하라
* `컨테이너 타입(Collection, 배열, 스트림 등)`은 `Optional`로 감싸지 말아라. 그냥 빈 객체을 반환하라.
* Optional 반환하는 메서드는 절대 `null`을 반환하지 말아라
* 기분타입을 반환하는 `Optional`을 사용하지 말아라.
    * `OptionalInt`, `OptionalLong`, `OptionalDouble`을 사용하라
* 클래스의 멤버변수로 `Optional`을 사용하는건 정말 드문 경우 아니면 비추다.
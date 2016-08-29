# Learning Typelevel
Typelevel Study using typelevel.org blog articles - http://typelevel.org/blog

## [Type members are [almost] type parameters](http://typelevel.org/blog/2015/07/13/type-members-parameters.html)
Type member and parameters 시리즈 1
```scala
// 원문 : http://typelevel.org/blog/2015/07/13/type-members-parameters.html

// type member
class Blar {
  type Member
}

// type parameter
class Blar2[Param]

// 위에 두개는 다른점보다는 유사한 점이 더 많다.

// type parameter가 대부분의 경우에 더 편하다. 타입을 existentially하게 쓰는 경우에는 아마 type member가 더 좋을것이다.

// 두개의 버전을 List를 구현을 통해서 비교해본다.
// 1. type parameter 버전
sealed abstract class PList[T]
final case class PNil[T]() extends PList[T]
final case class PCons[T](head: T, tail: PList[T]) extends PList[T]

// 2. type member 버전
sealed abstract class MList { self =>
  type T
  def uncons : Option[MCons { type T = self.T }]
}

sealed abstract class MNil extends MList {
  override def uncons = None
}

sealed abstract class MCons extends MList { self =>
  val head : T
  val tail : MList { type T = self.T }
  override def uncons = Some(self: MCons { type T = self.T })
}

// type parameter를 만들때 보다 훨 복잡하다. 그리고 아직 instance를 생성할수 있는 코드도 없다.

// 생성자 만들기, 함수를 통해 들어온 type parameter를 type member로 전달한다.
object MList {
  def MNil[T0](): MNil {type T = T0} =
    new MNil {
      type T = T0
    }

  def MCons[T0](hd: T0, tl: MList { type T = T0 }): MCons { type T = T0 } =
    new MCons {
      type T = T0
      val head: T0 = hd
      val tail: MList { type T = T0 } = tl
    }
}

// { type T = ... } 를 항상 달고 다녀야 하나?

// MCons의 tail에다가 정보를 type정보를 빼보자.
sealed abstract class MList2 { self =>
  type T
  def uncons : Option[MCons2 { type T = self.T }]
}

sealed abstract class MNil2 extends MList2 {
  override def uncons = None
}

sealed abstract class MCons2 extends MList2 { self =>
  val head : T
  val tail : MList2  // 여기에 type member 정보가 없앴다.
  override def uncons = Some(self: MCons2 { type T = self.T })
}

object MList2 {
  def MNil2[T0](): MNil2 {type T = T0} =
    new MNil2 {
      type T = T0
    }

  def MCons2[T0](hd: T0, tl: MList2 { type T = T0 }): MCons2 { type T = T0 } =
    new MCons2 {
      type T = T0
      val head: T0 = hd
      val tail: MList2 { type T = T0 } = tl
    }
}

// 그럼 테스트 해보자.
// 테스트는 TypeMemberVsTypeParameterSpec.scala에서 해봄.

// MCons2는 특정 동작에 대해서 컴파일 오류를 발생한다.
// MList라고 표현하는건 type parameter에서 PList[_]로 표현하는것과 같다.
// 함수형 언어에서는 이걸 existential이라 말한다. 용어자체가 잘 와 닿지는 않지만 자바의 List<?>의 와일드 카드와 같은 개념이라 생각하면 면된다.
// 트위터 스칼라 스쿨에도 이를 List[_], List[T forSome { type T }] 같은 표현을 와일드 카드라 표현하였다.
// 참조 : https://twitter.github.io/scala_school/ko/type-basics.html#quantification
// 한국어로 적절한 표현이 무엇일까? 고민....

// 지금은 용어 그래도 써보기로 한다. 고유명사처럼..

// 언제 existential이 OK? 괜찮은걸까?
// existential 버전으로 유용한 함수를 만들수 있다.

object Existential {
  // type member existential version
  def mlength(xs: MList2): Int =
    xs.uncons match {
      case None => 0
      case Some(c) => 1 + mlength(c.tail)
    }

  // type parameter version
  def plengthT[T](xs: PList[T]): Int =
    xs match {
      case PNil() => 0
      case PCons(_, t) => 1 + plengthT(t)
    }

  // type parameter existential version
  def plengthE(xs: PList[_]): Int =
    xs match {
      case PNil() => 0
      case PCons(_, t) => 1 + plengthE(t)
    }

  // existential 방식? 을 쓸 수 있는건 간단한 rule을 만족하면 된다.
  // 1. type parameter가 argument에 한번만 나타날때
  // 2. 그리고 result type에는 없을때.

  // mlength, plengthT, plenghtE의 테스트는 TypeMemberVsTypeParameterSpec.scala에서 또 해봄.

}
```

## [When are two methods alike?](http://typelevel.org/blog/2015/07/16/method-equiv.html)
Type member and parameters 시리즈 2
```scala
// 원문 : http://typelevel.org/blog/2015/07/16/method-equiv.html
// 지난 편에 이어 더 헷갈린다. existential 에 대해서 더 알아보는 시간이다. 
// 우선 컴파일이 안되는 아래의 코드가 있다. 될것 같은데 안된다.
def copyToZero(xs: ArrayBuffer[_]) : Unit = {
  xs += xs(0)   // 컴파일 안된다...
}
 Error:(8, 13) type mismatch;
 found   : (some other)_$1(in value xs)
 required: _$1(in value xs)
 xs += xs(0)

// 에러메시지가 당체 뭘 말하는지는 이해할수 어렵다. 모르겠다.

```

이번에 자바이다.
```java
   
public static void copyToZero(final List<?> xs) {
    xs.add(xs.get(0));  // 이것또한 컴파일이 안된다.
}

 Error:(8, 11) java: no suitable method found for add(capture#1 of ?)
 method java.util.Collection.add(capture#2 of ?) is not applicable
         (argument mismatch; java.lang.Object cannot be converted to capture#2 of ?)
 method java.util.List.add(capture#2 of ?) is not applicable
         (argument mismatch; java.lang.Object cannot be converted to capture#2 of ?)

// 이 또한 에러가 무슨말을 하는지 잘 이해되지 않는다.
// xs.get(0)한건 Object Type이고 java.util.List.add(? e) wildcard 타입이라 변형이 되지 않는다는 이야기 같다.
// 왜 xs.get(0)한건 object type인가? type erasure때문인가? wildcard 때문인가?
// wildcard type 이면 object type이라도 받아들여하는것 아닌가?
// 라는 궁금증만 남는다.
   
```

다시 스칼라
```scala
// 그러나 다행히도? 해결할수 있는 방법이 있다. 똑같이 타입 정보가 없는 existential이지만 
// type paramter가 있는 함수를 통해서 한번더 거쳐서 구현하면 잘 동작한다.
// Java와 Scala에는 existential을 method type parameter으로 lifting할수 있는 `equivalent` method type이 있다.

def copyToZeroE(xs: ArrayBuffer[_]): Unit =
  copyToZeroP(xs)

def copyToZeroP[T](xs: ArrayBuffer[T]): Unit = {
  val zv: T = xs(0)
  xs += zv
}
```

그러면 자바는?
```java
// 역시나 위와 같은 방법으로 구현하면 java도 잘된다. 
// 아래 코드는 이제 컴파일된다.
// Unit Test도 잘 통과한다. why???!!!!
public static void copyToZeroE(final List<?> xs) {
    copyToZeroP(xs);
}

public static <T> void copyToZeroP(final List<T> xs) {
    final T zv = xs.get(0);
    xs.add(zv);
}
```
그러면 위의 오류나던 코드를 다시 보자.
```scala
// 다시 컴파일 오류나던 copyToZero를 로컬 변수를 선언해서 다시 만들어보자.
def copyToZeroWithLV(xs: ArrayBuffer[_]): Unit = {
  val zv = xs(0)
//    xs += zv   // 역시나 컴파일 되지 않는다.
}

 Error:(37, 11) type mismatch;
 found   : zv.type (with underlying type Any)
 required: _$3
 xs += zv

// 변수에 대한 타입 추론이 전혀 도움이 되지 않는다.
// scala는 xs를 existential type으로 됨으로 간주하고 zv와 xs의 관계를 끊음으로서
// xs에서 독립적으로 zv의 정의를 만든다. 이로 인해 zv의 타입은 없어지고 Any로 추론된다.


// existential variant를 구현하기 위해서 type-parameterized variant를 호출했다.
// 이는 단지 equivalent method type를 이용하여 컴파일러에게 도움을 주었을뿐이다.

// 앞의 단순한 케이스에서 보았듯이, `scalac`과 `javac` 모두 추론을 관리하기 위해서 type `T`는 existential 이어야한다?
// 직접 쓸 수 없는 함수를 method equivalance와 generality는 아를 작성하고, 안전하게 만들어주는것이 가능하다.



// 뭔가 어려운 말이 계속 나옴... 영어 공부를 더 열심히 하자.

// wildcard는 잘못된 표현이다..

def pdropFirst[T](xs: PList[T]): PList[T] =
  xs match {
    case PNil() => PNil()
    case PCons(_, t) => t
  }

def mdropFirstT[T0](xs: MList { type T = T0 }): MList { type T = T0} = {
  import MList._
  xs.uncons match {
    case None => MNil()
    case Some(c) => c.tail
  }
}

// refinement를 잘라내보자. 컴파일이 되는듯 하다.
def mdropFirstE(xs: MList): MList = {
  import MList._
  xs.uncons match {
    case None => MNil()
    case Some(c) => c.tail
  }
}

// 확실히 더 깔끔하고 좋아보인다.
// 그런데 `mdropFirstE`는 `xs.T`를 type paramter로 `mdropFirstT`에 전달함으로서 이함수를 이용해 구현할수 있다.
def mdropFirstEUsingP(xs: MList): MList = {
  mdropFirstT[xs.T](xs)
}
// 그런데 반대로는 되지 않는다.
// `mdropFirstT` 는 <m `mdropFirstE` 혹은 mdropFirstT는 엄격하고 더 일반적이다.


// 아래 `mdropFirstE1`의 경우 type parameter 인자값 `T0`와 반환된는값 `Int`사이의 올바른 관계를 만드는것이 실패했다.
def mdropFirstE1[T0](xs: MList): MList = {
  import MList._
  MCons[Int](42, MNil())
}

// 강한 type 제약이 있는 `mdropFirstT`의 경우 이런 행위들을 금지하게 한다.

// 중간에 뭐라뭐라 어려운 말 나옴... ㅡㅡ;; 다시한번 영어공부를 더 열심히 해야겠다 생각했다.

def goshWhatIsThis[T](t: T): T = null  // 컴파일 되지 않음
 Error:(105, 36) type mismatch;
 found   : Null(null)
 required: T

```

// 그러나 자바버전은 된다.
```java

// 스칼라에서 컴파일 안되는 코드가 java에서는 가능하다.
public static <T> T holdOnNow(T t) {
    return null;
}
```

그리고 자바버전을 holdOnNow를 이용해서 스칼라의 goshWhatIsThis을 구현할수 있다.
```scala

def goshWhatIsThis1[T](t: T): T = MethodEquivalenceJava.holdOnNow(t)

// 그리고 반대로?도 된다 그러면 그들은 equivalent 이다. 그러나 타입이 null은 반환을 할수 없다한다.

// 자바의 generic type은 class type만 젹용될수 있기 때문에 이문제로 자바는 암묵적으로 upper bound를 넣는다.
// scala에서는 `[T <: AnyRef]`와 같은 의미가 된다.
// 만약 이 제약을 scala에 넣는다면 에러가 나온다.

def holdOnNow[T <: AnyRef](t: T): T = MethodEquivalenceJava.holdOnNow(t)
def goshWhatIsThis2[T](t: T): T = holdOnNow(t)
 Error:(124, 37) inferred type arguments [T] do not conform
 to method holdOnNow's type parameter bounds [T <: AnyRef]

// 아 어렵다... 나중에 다시 읽고 정리해봐야겠다.
// 컴파일러 에러만 계속 보여주고...
```
3부족이다. 영어, 함수형 언어, 스칼라 이해도 부족이다. :sob: 


## [What happens when I forget a refinement?](http://typelevel.org/blog/2015/07/19/forget-refinement-aux.html)
Type member and parameters 시리즈 3

```scala
// refinement를 뺐을때 무엇이 일어날까?

// 앞에서 `MList`를 가지고 에러를 내면서 했다면 이번엔 `PList`를 가지고 해보자.

def pdropFirst(xs: PList) = ??? // 컴파일 안됨

Error:(9, 22) class PList takes type parameters

// PList에 type parameter가 없기 때문에 컴파일 오류가 발생한다.

// 또다른 의도적인 실수를 만들어보자.
// 앞에서 type parameter, member를 T로 선언했다.
// 이는 자바에서는 일반적인 컨벤션이지만 스칼라에서는 일반적이지 않다.
// 스칼라에서는 관용적으로 A를 이용해서 type parameter나 member를 표현한다. F[A], F[A, B] 등등등..

def mdropFirstE2[T0](xs: MList { type A = T0 }) = {
  import MList._
  xs.uncons match {
    case None => MNil()
    case Some(c) => c.tail
  }
}

// 위의 코드는 컴파일은 된다. 그러나 실행을 하면...

mdropFirstE2(MNil[Int]())
Error:(12, 29) type mismatch;
found   : MNil{type T = Int}
required: MList{type A = ?}

// 타입이 맞지 않다는 에러가 발생한다.

// MList는 type member A가 없다.
// `MList {type A = T0}`에서 type member A은 다른 trait에서 MList로 mixin되거나 inner subclass 으로 부터 올수 있다.
// 어떤것은 sealed 속성이나 final 가진 것들은 instant화 될수 없고 아니면 그외의 것은 부적절한? 것이 된다.

// 값(value)가 없는 타입들은 자바와 스칼라 양쪽 모두 의미가 있고 유용하다.

// # 왜 T0? Aux가 뭔가?

// `MList`의 몇몇 함수는 `T`대신 `T0`를 이용하여서 type parameter를 취해왔다.
// 이것은 기억, 메모의 방법으로 `scalac`가 나에게 허락하면 나는 여기서 `T`를 사용하겠다고 미리 서술하는 것이다.
// 이것은 scalaz.Unapply에서 사용한 방식을 차용했다.
// https://github.com/scalaz/scalaz/blob/v7.1.3/core/src/main/scala/scalaz/Unapply.scala#L217

// `T0`대신 `T`를 사용해보자.
// T에 대한 순한 참조가 일어나면서 컴파일 되지 않는다.
def MNil3[T]() : MNil { type T = T} =
  new MNil {
    type T = T
  }
Error:(51, 36) illegal cyclic reference involving type T

// 이것은 scoping 문제이다. refinement type이 member `T`가 method type parameter `T`를 가리게 만든다.
// 이문제를 `MCons#uncons`와 `MCons#tail`에서 다루었다. 이 케이스에는 외부 `T`를 `self.T`로 대체하였다.

// type를 member와 같이 정의 할때 member를 type parameter로 바꾸기 위한 `Aux` type을 companion에 정의 해야한다.

// `Aux`의 이름은 shapeless의 conversion에서 빌려왔다.

// `object MList` 이것들을 추가해보자.
object MList {
  type Aux[T0] = MList { type T = T0 }
}

// 이제 `MList { type T = Int }` 대신에 `MList.Aux[Int]`로 쓸수 있다. 깔끔하다.
// mdropFirstT는 새로운 스타일로 함수를 다시 쓸수 있다.
def mdropFirstT2[T](xs: MList.Aux[T]) : MList.Aux[T] = ???

// 익숙하진 않지만 훨씬 더 세련되었다.
// 또한 member `T`는 `Aux`의 type parameter의 위치에 대한 범위가 아니므로
// 에러없이 type parameter의 이름은 `T`로 명명하고 `MList.Aux[T]`라고 쓸쑤 있다.
// 이것은 type parameter의 장점이고, `PList`는 처음부터 이런 문제가 없었다.

// `Aux`를 사용함으로서 type member를 명시하는것을 빼먹거나 오타로 인한 오류 발생을 피하는것을 도와줄것이다.
```




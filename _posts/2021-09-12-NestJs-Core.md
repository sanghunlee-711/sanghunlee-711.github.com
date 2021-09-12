---
title: Nest js 로 유저단 만들기[0]
author: Sanghun lee
date: 2021-09-08 11:33:00 +0800
categories: [Blogging, NestJs]
tags: [NestJs]
math: true
mermaid: true
image:
  src: https://nestjs.com/img/nest-og.png
  width: 850
  height: 585
---

# Nest js의 기본적인 개념

## 서론

NestJs는 기본적으로 Express위에서 돌아가는 FrameWork다.
프론트엔드 개발자로 개발커리어를 시작한 내가 NestJs를 고른 이유는 백엔드에 대한 이해도가 조금 낮은 상태이기에 우선 무엇인가를 만들면서 전체적인 흐름을 파악하고 싶어서인것이 첫번째 이유였다.(그리고 타입스크립트 완벽지원도 한몫 했다.)

두번째는 개발 시작과 함께했던 노마드코더에 마침 좋은 강의가 있어서이다 :)..

우선, 시작을 위해서 공식문서를 통해 간단하게 설치를 진행하였고 graphql, postgreql, typeorm을 추가적으로 사용하였다.

이 과정에서 의존성 문제로 인해 몇번이나 force를 하였다 ㅎㅎ...(노드의존성 진짜 ..)

이 시리즈에서 이 편은 간단하게 내가 구성한 방식의 개념에 대해 설명하고, 다음 장 부터는 로직을 조금 뜯어보며 언젠가 보실 분들을 위해(&&나의 이해를 위해) 도움이 되고자 추가적인 설명을 곁들이며 진행해보겠다.

## Nest Js의 코어 개념

> 대부분 Nest Js공식문서의 말을 재 반복하는 내용이므로 관심없으신분들은 빠르게 이 파트를 스킵하시면 됩니다

일단 나는 RestAPI 방식이 아닌 graphql을 통해 진행하였기 때문에 @Get, @Post와 같은 데코레이터를 사용하지 않아도 되었다.

데코레이터는 클래스, 메서드 또는 속성에 대해 정의하는 기능을 하게 되며 nestjs에서 기본적으로 다양한 데코레이터를 제공해준다

이로 인해 개발속도가 폭발적으로 증가하게 되는 경험을 하였다 ㅋ..

크고 간단하게 몇가지 개념들만 설명하고 내가 구현한것들을 정리하면서 세세한 부분을 푸는게 좋을 것 같다.

크게 module, resolver, service라는 개념으로
각 모듈은 기본적으로 프로바이더를 캡슐화한다.

![nestjsofficial-module](https://docs.nestjs.kr/assets/Modules_1.png)
_module-concept_

그럼 프로바이더란 무엇이냐 Nest 인젝터에 의해 인스턴스화 되고 적어도 이 모듈에서 공유될 수 있는 것을 말하는 것이라고한다..

말로 설명하기 어려우니 간단한 샘플을 보며 설명을 하는 것이 좋을 것 같다.

```typescript
@Module({
  imports: [TypeOrmModule.forFeature([User, Verification])],
  providers: [UsersResolver, UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

_users.module.ts_

위는 `users.module.ts` 파일의 코드이다.

일단 `app.module.ts`(모든 모듈의 루트 모듈이라고 보면 된다-> TMI로 Nest js는 `싱글톤 패턴`을 가지고 있다)에서 `TypeOrmModule` 대한 정의를 내릴 때 entity에 user와 verification이 등록된 것을 가져오기 위해 imports메서드를 사용한 것을 볼 수 있다.

이로 인해 자연스럽게 유추할 수 있는것은 다른 모듈 또는 루트 모듈에 존재하는 것을 해당 모듈에서 사용하기 위해서는 그 모듈의 `imports 메서드`를 사용해야한다는 것이다.

그리고 `provider`는 위에서 언급하였듯 해당 모듈 내에서 공유될 수 있는 것들에 대한 정의를 내려주는 곳인데 userResolver와 userService가 있는 것을 볼 수 있다.

여기서 `resolver`의 역할은 GraphQL operation(query, mutation, subscription)을 데이터로 변환하기 위한 명령을 제공하기 위해 사용된다.
자세히 보면 graphql을 통해 어떠한 인자를 받을 것인지에 대한 정의가 되어있는 dto파일을 통해 인자를 받고 해당 로직이 완성된 후 return할 타입에 대한 정의가 되어있다.

추가로 이 로직이 Mutation인지 Query인지에 대한 정의도 한다.

```typescript
  @Role(['Any'])
  @Mutation(() => EditProfileOutput)
  async editProfile(
    @AuthUser() authUser: User,
    @Args('input') editProfileInput: EditProfileInput,
  ): Promise<EditProfileOutput> {
    return this.usersService.editProfile(authUser, editProfileInput);
  }
```

_users.resolver.ts 중 일부_

그리고 `service`의 역할은 해당 resolver가 실행시키고 싶은 비즈니스 로직 부분을 담당하게 된다.

아래 코드는 위의 `users.resolver.ts 중 일부`에서 return 시킨 서비스 로직이다.

```typescript
  async changeRole({ role, id }: ChangeRoleInput): Promise<ChangeRoleOutput> {
    try {
      const user = await this.users.findOne({ id });

      if (!user) {
        return {
          ok: false,
          error: '존재하지 않는 유저 입니다..',
        };
      }

      user.role = role;

      this.users.save(user);

      return {
        ok: true,
      };
    } catch {
      return {
        ok: false,
        error: '알 수 없는 이유로 권한변경에 실패하였습니다.',
      };
    }
  }
```

_users.service.ts 중 일부_

이렇게 큰 틀에서 보면 여러개의 module+resolver+service의 합이 app.module에서 합쳐져 안전하게 운영될 수 있게 된다.

위에서 보이는 @Args('input)이나 @Role()데코레이터는 자세한 구현사항을 설명하며 함께 이야기 해보겠다 :)

일단 기본적인 틀에 대한 개념은 아주 간략하게 설명이 된 것 같다.

> PS. 해당 user단의 구성방식은 [(풀스택) 우버 이츠 클론코딩](https://nomadcoders.co/nuber-eats/lobby)의 강의에서 나오는 방식과 매우 유사하게 진행되었다.

> 추가적으로 내가 만들고 싶은 서비스(커뮤니티)에 필요한 몇개의 mutation과 query들만 추가 되는 내용으로 설명이 될 것이므로 해당 강의를 보신분은 빠르게 강의를 다시보시는 것도 추천한다 ㅎㅅㅎ..

## 참고

- [Exploring EcmaScript Decorators](https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841)
- [docs.nestjs.kr/custom-decorators](https://docs.nestjs.kr/custom-decorators)
- [github-nomadcoder/nuber-eats-backend](https://github.com/nomadcoders/nuber-eats-backend)

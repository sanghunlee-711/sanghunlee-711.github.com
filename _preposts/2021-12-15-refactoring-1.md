---
title: 리액트에서 비즈니스 로직을 나눠보자[1 - 비즈니스 로직]
author: Sanghun lee
date: 2021-12-15 11:33:00 +0800
categories: [FE, React]
tags: [Architecture]
math: true
mermaid: true
image:
  src: https://layer.team/wp-content/uploads/2020/04/layer-app-spot-26-best-field-data-software-revit-addin-manage.png
  width: 850
  height: 585
---

# 왜 하게 되었냐면

회사에서 개발자 다수가 이탈하게 되면서 빠른 유지보수와 코드의 정확성을 높이는 일이 나에게도, 회사에게도 중요해졌다.

현재 리액트 프로젝트에서 Toolkit과 Saga를 통해서 나름 나눈다고 나눴지만 이 과정에서 불필요한 코드들도 많아졌고,

실제로 state나 hooks와 연결된 부분은 컴포넌트에 그대로 남아있게 되었다. 그래서 리팩토링이 필요한 상황이었다.

마침 앵귤러로 진행된 다른 프로젝트를 조금씩 리액트로 포팅하는 와중이라 앵귤러의 구현 개념을 맛보고 즐기고 있었는데,

Nest JS에서 했던 방식과 유사하게(역시 같은팀이 맞다. 공식문서 말대로 똑같음) Service와 View가 거의 나눠진 상태였고

리액트에서도 이와 같은 방식은 아니더라도 로직과 뷰의 분리는 필요하다고 판단되었다.

크게 진행한 이유를 두가지 정도 들 수 있다.

1. 테스트 코드 짜는데 너무 힘들다

   - 예를 들어 Jest를 통해 해당 앱의 뷰를 테스트 시키는 것만해도 모든 함수들에게 적절하게 state가 들어가게 선언된 hook등을 모두 넘겨주어야했다.
   - 비즈니스로직과 뷰를 나눌 수 있게 된다면 로직을 위한 테스트와 뷰를 위한 테스트를 별도로 짤 수 있게 되므로 뷰만 변경 된 경우 로직쪽 테스트 파일은 그대로 남겨둬도 되는 장점이생긴다.
     - 물론 하나의 파일로 전체를 테스트해도 되지만 여간 긴것도 문제고 보기가 힘들다 .. (눈이 아프다.)

2. 유지보수 할 때 마다 짜증이 난다.

   - TSX 파일을 열때마다 무수한 비즈니스 로직이 연결되어있는 그 스크롤을 내려야 비로소 내가 간단하게 수정할 태그들이 보이는 것이 불편하다.
     - 가령 state하나 간단하게 선언해서 모달만 초기 렌더시에 띄워주면 되는 간단한 상황에서도 해당 파일 전체가 수정되는 것이 별로다.
   - 그냥 코드가 못생겨보인다.

그리고 퇴사자들의 여러가지 기존 코드를 CTO님께서 뜯어보시고 프론트쪽에 많은 의문을 품으셔서 이왕 사람도 적어진 김에 많은 질문을 드렸고 수확이 컸다.

1. 앱(웹 어플리케이션)에서의 작은 규모인 경우 레이어를 나누는 것이 더 복잡해 보이는 경우가 있어서 개발초기에는 레이어의 나눔이 무의미할 수도 있다.
2. 규모가 큰 경우 유지보수 및 테스트 코드 작성의 효율(진짜 이건 작성해보면 뼈져리게 느껴진다.)을 위해서는 나누는 것이 굉장히 효율적이다.
3. 새로운 사람이 해당 코드를 봤을 때 구조를 통해서 이해가 용이해질 수도 있다.
4. 지금 리팩토링을 진행하면서 한번에 어떤구조를 정해서 적용하는 것이 아니라 하나씩 필요한 부분을 분리해 나아가면서 부족한 개념을 찾아서 공부하다보면 패턴이 자연스럽게 공부가 된다
   - 이제 끝까지 다 쪼개면 Atomic Pattern이 되는 것..

이제껏 요청한 UI, 기능들을 일정 무리 없이 뽑아내었던 상태라 크게 코드쪽을 신경쓰시지 않으셨었는데(인원수도 적고 해야할 일의 양자체도 많았었다.)

이제는 좀 리뷰를 해주시고 도움을 주시겠다고 하셨다 ㅎㅎ..(많이 심각한가 보다.)

그래서 리팩토링하기에 정말 좋은 재료로서의 코드라고 이야기해주셔 그 여정을 나름 포스팅으로 담아보려고 작성을 하게 되었다.

# Business Logic

일단 비즈니스 로직의 정의를 컴포넌트 내부에서 이루어지는 Actions로 정의했다.

가령 아래코드의 `onApply` 같은 코드가 될 수 있다.

글로벌 스테이트를 사용하고 다양한 로직을 통해 state를 변경할 액션을 dispatch한다거나, 비동기 처리로 데이터를 불러오는등의 역할을 하는 것들이다.

```tsx
export default function FirstSection() {
  const history = useHistory();
  const dispatch = useDispatch();
  const { authPhase, mainPopup, companyInfoData } = useRootState(
    (state) => state.global,
  );
  const { basicInfoTitle, check, origin, path } = personnelManagement;
  const { corpName, charger, chargerTel, address, addressDetail } =
    companyInfoData;

  const onApply = () => {
    if (authPhase === 'logout' && !localStorage.getItem('access_token')) {
      return dispatch(setIsLoginModalVisible(true));
    }

    dispatch(
      checkBasicCompanyInfoAction({
        check: check as (keyof ICompanyInfo)[],
        domain: origin,
        path,
      }),
    );
  };

  useEffect(() => {
    if (localStorage.getItem('isPaused') === 'true') {
      localStorage.removeItem('isPaused');
      onApply();
    }
  }, []);

  return (
    <>
      <FirstSectionWrapper>
        <HeaderTextWrapper>
          <AText color="black100" size={45} bold>
            비즈니스로직으로
          </AText>
          <AText color="black100" size={45} bold>
            분리를 제발하자
          </AText>
        </HeaderTextWrapper>
        <FirstSectionButtonWrapper>
          <ABasicButton
          />
          <ABasicButton
            onClick={onApply}
          />
        </FirstSectionButtonWrapper>

    </>
  );
}

```

그래서 위와 같은 코드를 아래와 같이 파일을 분리하여 개선하는 방향을 생각할 수 있다.

컴포넌트만 놓고 보았을때, 일단 로직을 분리할 수 있게 되어 필요한 state나 actions만 Service에서 가져와 선언 또는 호출을 하면 된다.

다만 이 상태에서 아쉬운 점은 뷰가 완전히 분리되었다고 볼 수 없는 상태이다.

예를 들어 FirstSection이라는 컴포넌트가 테스트에 들어가게 된다면,

`PersonnelManagementComponentsService`를 가지고와서 선언을 해준 뒤 테스트를 해줘야하는데

명확히 뷰의 역할만 하는 것이 아니라고 판단이 되기 때문이다.

그래서 추후에 뷰는 Props로 필요한 로직들만 넘겨 처리하는 방식으로 진행하고 테스트에서는 이 Props를 유연하게 정해줄 수 있으므로 이를 활용할 생각이다.(아직까진..)

```tsx
export default function FirstSection() {
  const { actions, state } = PersonnelManagementComponentsService();

  useEffect(() => {
    if (localStorage.getItem('isPaused') === 'true') {
      localStorage.removeItem('isPaused');
      actions.onApply();
    }
  }, []);

  return (
    <>
      <FirstSectionWrapper>
        <HeaderTextWrapper>
          <AText color="black100" size={45} bold>
            비즈니스로직으로
          </AText>
          <AText color="black100" size={45} bold>
            분리를 제발하자
          </AText>
        </HeaderTextWrapper>
        <FirstSectionButtonWrapper>
          <ABasicButton
          />
          <ABasicButton
            onClick={actions.onApply}
          />
        </FirstSectionButtonWrapper>

    </>
  );
}
```

아래는 분리한 서비스 로직이다.
사실 그냥 복붙한 것과 같은 것이다(아직은 ..)

```ts
const PersonnelManagementComponentsService = () => {
  const history = useHistory();
  const dispatch = useDispatch();
  const { authPhase, mainPopup, companyInfoData } = useRootState(
    (state) => state.global
  );

  const {
    address: addressPopup,
    certificate: certificatePopup,
    personnelManagement,
  } = MAIN_POPUP;
  const { basicInfoTitle, check, origin, path } = personnelManagement;

  const onApply = () => {
    if (authPhase === "logout" && !localStorage.getItem("access_token")) {
      return dispatch(setIsLoginModalVisible(true));
    }

    // 로그인이 되었으면
    dispatch(
      checkBasicCompanyInfoAction({
        check: check as (keyof ICompanyInfo)[],
        domain: origin,
        path,
      })
    );
  };

  return {
    state: {
      mainPopup,
      ...
    },
    actions: {
      onApply,
      ...
    },
  };
};

export default PersonnelManagementComponentsService;
```

이렇게 컴포넌트 내부에 선언되었던 함수나 state를 다른 파일로 넘겨온 뒤 `state`, `actions` 로 별도 제공을 해주고

컴포넌트 내부에서는 필요한 로직과 state만 가져오게 만드는 것이다.

### 문제점

렌더링의 최적화 및 테스트 코드를 작성 후에 추가 작성을 해보자.

## 참고

- [How to Create a Three-Layer Application with React Hooks](https://medium.com/dailyjs/how-to-create-a-three-layer-application-with-react-hooks-2efba9ba345f)
- 별도의 외부코드

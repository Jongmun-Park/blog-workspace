---
title: Jest error handling, .rejects function
date: "2019-11-06"
template: "post"
draft: false
slug: "/posts/3"
category: "Javascript"
tags:
  - "Jest"
  - "Javascript"
description: ""
---

jest 에러 테스트를 만들다가 컨디션이 어떻게 된 건지 굉장히 헷갈리는 부분이 생겨서 정리를 해두려 한다.

```js
/* Errors can be handled using the .catch method.
Make sure to add expect.assertions to verify that a certain number of assertions are called.
Otherwise a fulfilled promise would not fail the test: */

// using async/await.
it('tests error with async/await', async () => {
  expect.assertions(1);
  try {
    await user.getUserName(1);
  } catch (e) {
    expect(e).toEqual({
      error: 'User with 1 not found.',
    });
  }
});
```

출처 : https://jestjs.io/docs/en/tutorial-async#error-handling

jest 공식 문서에 나와있는 error handling 설명이다.
내가 헷갈렸던 부분은 `expect.assertions(1);` 이 코드이다.
> Make sure to add expect.assertions to verify that a certain number of assertions are called. Otherwise a fulfilled promise would not fail the test

설명을 보면 반드시 assertions 코드를 추가하라고 나와 있다.
저게 무슨 말일까.. 이해가 가질 않았다.

예제 코드를 돌려보고 이해할 수 있었다.

```ts
describe.only('buildWhereOptionsFromFiltersForExamQuestion2', () => {
  test('throw when examID is invalid', async () => {
    expect.assertions(1)
    const userModel = await createSuperUser()
    const { examService } = createServices(sequelize, userModel)
    try {
      await examService.buildWhereOptionsFromFiltersForExamQuestion({ examID: 1 })
    } catch (error) {
      expect(error).toEqual(new UserInputError('invalid examID'))
    }
  })
})
```

이 테스트는 examID가 유효하지 않을 때 발생시키는 에러를 테스트하는 것이다.
근데 유효한 examID '1'을 부여하면 ?

![](/static/media/buildWhereOptionsFromFiltersForExamQuestion2.png)

이러한 에러가 발생한다.
"너가 의도한 one assertion이 아닌 zero assertion이 돌아왔다."
만약 assertions 코드가 없었더라면 우리는 내가 의도했던 에러가 발생했는지 안했는지 알 방도가 없다.

예를 들어 assertions 코드가 없고, 에러를 발생시키지 않는 유효한 값이 들어가면 try 구문만 거치고 저 테스트는 끝났을 것이다. 에러 테스트를 제대로 하지 못하고 있는 것이다. 우리가 원했던 건 "내가 예상했던 에러가 제대로 발생하는가"를 확인하는 것이다.

그렇다면 .rejects 함수는 어떨까?

```ts
describe.only('buildWhereOptionsFromFiltersForExamQuestion1', () => {
  test('throw when examID is invalid', async () => {
    const userModel = await createSuperUser()
    const { examService } = createServices(sequelize, userModel)
    await expect(
      examService.buildWhereOptionsFromFiltersForExamQuestion({ examID: 1 }),
    ).rejects.toEqual(new UserInputError('invalid examID'))
  })
})
```

assertions 코드를 추가하지 않아도 된다.
그 이유는 rejects 함수 안에 assertions 기능이 내재되어 있기 때문이다.
assertions 코드 없이 유효한 examID 값인 '1'을 부여해도 rejects 함수는 "너가 예상한 에러가 발생하지 않았다"는 것을 알려준다.

![](/static/media/buildWhereOptionsFromFiltersForExamQuestion1.png)

"Received promise resolved instead of rejected" 불합격으로 처리된 것이 아닌 해결된 promise가 돌아왔다는 것이다.
에러(불합격)를 예상(expect)했지만 말이다.

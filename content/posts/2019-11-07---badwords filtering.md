---
title: badwords 필터링(작성중)
date: "2019-11-07"
template: "post"
draft: false
slug: "/posts/4"
category: "Javascript"
tags:
  - "bad words"
  - "Javascript"
description: ""
---


#### profane words & bad words 관련 오픈소스
1. https://github.com/web-mech/badwords
2. https://github.com/ChaseFlorell/jQuery.ProfanityFilter.git
3. https://github.com/dariusk/wordfilter

1, 2, 3번 오픈소스의 코드를 살펴본 결과,

2번 코드는 old style 코드라고 판단이 들었다.
3번은 javascript와 python 버젼 중 하나를 선택해야 한다.
요즘 javascript를 연습하는 김에, 그리고 회원가입 입력값 관련 validation은 프론트 단에서 처리하는 것이 낫다고 생각하여
이 작업은 javascript로 하기로 결정.

그렇다면 1번과 3번 중 하나를 선택해야 했다.

#### 1번 코드

```js
const localList = require('./lang.json').words;
const baseList = require('badwords-list').array;

class Filter {

  /**
   * Filter constructor.
   * @constructor
   * @param {object} options - Filter instance options
   * @param {boolean} options.emptyList - Instantiate filter with no blacklist
   * @param {array} options.list - Instantiate filter with custom list
   * @param {string} options.placeHolder - Character used to replace profane words.
   * @param {string} options.regex - Regular expression used to sanitize words before comparing them to blacklist.
   * @param {string} options.replaceRegex - Regular expression used to replace profane words with placeHolder.
   * @param {string} options.splitRegex - Regular expression used to split a string into words.
   */
  constructor(options = {}) {
    Object.assign(this, {
      list: options.emptyList && [] || Array.prototype.concat.apply(localList, [baseList, options.list || []]),
      exclude: options.exclude || [],
      splitRegex: options.splitRegex || /\b/,
      placeHolder: options.placeHolder || '*',
      regex: options.regex || /[^a-zA-Z0-9|\$|\@]|\^/g,
      replaceRegex: options.replaceRegex || /\w/g
    })
  }

  /**
   * Determine if a string contains profane language.
   * @param {string} string - String to evaluate for profanity.
   */
  isProfane(string) {
    return this.list
      .filter((word) => {
        const wordExp = new RegExp(`\\b${word.replace(/(\W)/g, '\\$1')}\\b`, 'gi');
        return !this.exclude.includes(word.toLowerCase()) && wordExp.test(string);
      })
      .length > 0 || false;
  }
```

#### 3번 코드

```js
/*
 * wordfilter
 * https://github.com/dariusk/wordfilter
 *
 * Copyright (c) 2013 Darius Kazemi
 * Licensed under the MIT license.
 */

'use strict';

var blacklist, regex;

function rebuild() {
  regex = new RegExp(blacklist.join('|'), 'i');
}

blacklist = require('./badwords.json');
rebuild();

module.exports = {
  blacklisted: function(string) {
    return !!blacklist.length && regex.test(string);
  },
  addWords: function(array) {
    blacklist = blacklist.concat(array);
    rebuild();
  },
  removeWord: function(word) {
    var index = blacklist.indexOf(word);
    if (index > -1) {
      blacklist.splice(index, 1);
      rebuild();
    }
  },
  clearList: function() {
    blacklist = [];
    rebuild();
  },
};
```


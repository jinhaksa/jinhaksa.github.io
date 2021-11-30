---
layout: post
title: Vue3에서 Composition API 이해하기
categories: [dev]
tags:
  - vue
  - nodejs
  - typescript
author: 심동근
email: sdg@jinhakapply.com
date: 2021-11-30
---

# 들어가기 앞서

Vue3의 Composition API에 대한 공식 문서를 읽었는데 한 번에 이해가 되지않았다. 그래서 해당 내용을 설명 및 주관적 의견을 붙여 재작성했다.

# Composition API란

Vue2에서 컴포넌트 작성시 우리는 여러 기능들을 Vue의 API 옵션들(data, method, watch 등)에 나눠서 작성했다. 한 기능에 대한 소스코드가 이곳 저곳에 분리되어있는 것이다. 그럼으로써 소스코드 이해가 어려지고 유지보수 및 코드 재사용이 힘들어졌다.

Composition API는 소스코드를 기능별로 응집시킬 수 있게 된다. 그러므로 어떤 기능에 대한 소스코드 분석시 해당 기능이 작성되있는 소스코드 뭉텅이만 보면된다.

# Vue2에서의 소스코드

아래는 유저 레포지토리에서 리스트를 출력하는 컴포넌트이다. 그리고 기능으로는 검색 및 필터가 있다.

```javascript
// src/components/UserRepositories.vue

export default {
  components: { RepositoriesFilters, RepositoriesSortBy, RepositoriesList },
  props: {
    user: { type: String }
  },
  data () {
    return {
      repositories: [], // 1
      filters: { ... }, // 3
      searchQuery: '' // 2
    }
  },
  computed: {
    filteredRepositories () { ... }, // 3
    repositoriesMatchingSearchQuery () { ... }, // 2
  },
  watch: {
    user: 'getUserRepositories' // 1
  },
  methods: {
    getUserRepositories () {
    }, // 2
    updateFilters () { ... }, // 3
  },
  mounted () {
    this.getUserRepositories() // 1
  }
}
```

소스코드를 보면 아래의 한 기능에 대한 소스가 분산되있는 것을 확인할 수 있다.

1. 리스트
2. 필터
3. 검색

추후에 기능이 더욱 많아지고 해당 기능에 관한 소스코드를 찾으려면 이해하기 힘들고 시간이 걸릴 것이다.

# Composition API 사용법

위에서 우리는 Vue3에서 소스코드를 한곳에 모아서 작성할 수 있다는 것을 알았다.

이제 아래 그림처럼 소스코드를 기능별로 모으면 되는 것이다.

![](/assets/img/posts/dev/2021-11-30-vue3-composition-api/0.png)

## setup(props, context)

setup 함수는 Composition API를 작성하는 곳이다. 기존에 data, method, created 등 나눠 담았던 소스코드를 setup 함수 안에 전부 넣을 수 있다.

setup 함수는 두 가지 파라미터가 있다.

1. props
2. context

### props

setup 함수의 props는 반응성이 있다. 한 마디로 props 값이 변경되면 화면이 업데이트가 되는 것이다.

아래의 소스코드에 Vue2 소스코드의 "1" 주석이 있는 리스트 기능 부분을 작성하였다.

```javascript
// src/components/UserRepositories.vue
import { fetchUserRepositories } from '@/api/repositories'

setup (props) {
  let repositories = []

  const getUserRepositories = async () => {
    repositories = await fetchUserRepositories(props.user)
  }

  return {
    repositories,
    getUserRepositories
  }
}
```

위 소스코드에 repositories 변수는 반응성이 없다. 즉 값이 변해도 화면이 업데이트가 되지 않는다.

## ref

Vue3에서는 ref를 사용해 어디에서나 반응성이 있는 변수를 만들 수 있게 했다.

```javascript
import { ref } from "vue";

const counter = ref(0);

console.log(counter); // { value: 0 }
console.log(counter.value); // 0

counter.value++;
console.log(counter.value); // 1
```

ref의 값은 객체가 감싸고 있으며 값은 ref변수의 value 키에서 확인할 수 있다. 또한 값을 확인하거나 변경할 때에는 value에 접근해야한다.

값을 감싸는 이유는 javascript 에서 Number와 String과 같은 타입은 참조에 의한 전달(pass by reference)이 아니라 값에 의한 전달(pass by value)이기 때문이다.
![](/assets/img/posts/dev/2021-11-30-vue3-composition-api/1.gif)

아까 작성한 소스코드에서 repository 변수를 반응성 있게 수정해보자.

```javascript
// src/components/UserRepositories.vue
import { fetchUserRepositories } from '@/api/repositories'
import { ref } from 'vue'

setup (props) {
  // ref 변수 생성
  let repositories = ref([])

  const getUserRepositories = async () => {
    repositories.value = await fetchUserRepositories(props.user)
  }

  return {
    repositories,
    getUserRepositories
  }
}
```

이제 getUserRepositories 함수를 호출하면 repositiory는 변경되고 화면이 업데이트 될 것이다.

## 라이프사이클 훅(콜백 함수) 등록

기존에 사용하던 라이프사이클 훅을 setup 함수에서 등록할 수 있다.
예를 들어 mounted 함수이면 앞에 접두사 on을 붙여서 onMounted 함수를 사용하면 된다.

```javascript
// src/components/UserRepositories.vue
import { fetchUserRepositories } from '@/api/repositories'
import { ref, onMounted } from 'vue'

// 컴포넌트 내부
setup (props) {
  const repositories = ref([])
  const getUserRepositories = async () => {
    repositories.value = await fetchUserRepositories(props.user)
  }

  onMounted(getUserRepositories) // mounted에서 getUserRepositories 호출

  return {
    repositories,
    getUserRepositories
  }
}
```

## watch

기존에 사용하던 watch 처럼 vue에서 import한 watch 함수로 변수를 감시할 수 있다.

watch에는 3가지 인자가 있다. `watch(source, callback, watchOptions)`

- 감시를 원하는 반응성 참조나 Getter 함수 (source)
- (value, oldValue, onInvalidate) => void 형태의 콜백 (callback)
- immediate나 deep과 같은 옵션들 (watchOptions)

```javascript
import { ref, watch } from "vue";

const counter = ref(0);
watch(counter, (newValue, oldValue) => {
  console.log("새로운 counter 값: " + counter.value);
});
```

이제 이전 소스코드에서 props의 user가 변할 때를 감시하고 변하면 fetch 함수를 실행해보자.

```javascript
// src/components/UserRepositories.vue
import { fetchUserRepositories } from '@/api/repositories'
import { ref, onMounted, watch, toRefs } from 'vue'

// 컴포넌트 내부
setup (props) {
  // `props.user`에 참조 .value속성에 접근하여 `user.value`로 변경
  const { user } = toRefs(props)

  const repositories = ref([])
  const getUserRepositories = async () => {
    // `props.user`의 참조 value에 접근하기 위해서 `user.value`로 변경
    repositories.value = await fetchUserRepositories(user.value)
  }

  onMounted(getUserRepositories)

  // props로 받고 반응성참조가 된 user에 감시자를 세팅
  watch(user, getUserRepositories)

  return {
    repositories,
    getUserRepositories
  }
}
```

## computed

computed는 함수의 연산된 결과 값을 반응성 있게 사용할 수 있다.

```javascript
import { ref, computed } from "vue";

const counter = ref(0);
const twiceTheCounter = computed(() => counter.value * 2);

counter.value++;
console.log(counter.value); // 1
console.log(twiceTheCounter.value); // 2
```

이제 coumted를 이용하여 "2" 주석이 있는 검색 기능 부분 작성해야한다.

```javascript
// src/components/UserRepositories.vue
import { fetchUserRepositories } from '@/api/repositories'
import { ref, onMounted, watch, toRefs, computed } from 'vue'

// 컴포넌트 내부
setup (props) {
  const { user } = toRefs(props)

  const repositories = ref([])
  const getUserRepositories = async () => {
    repositories.value = await fetchUserRepositories(user.value)
  }

  onMounted(getUserRepositories)

  watch(user, getUserRepositories)

  // 검색어 searchQuery 반응성 변수 생성
  const searchQuery = ref('')
  // computed 생성
  const repositoriesMatchingSearchQuery = computed(() => {
    return repositories.value.filter(
      repository => repository.name.includes(searchQuery.value)
    )
  })

  return {
    repositories,
    getUserRepositories,
    // search 관련 변수 및 함수 반환
    searchQuery,
    repositoriesMatchingSearchQuery
  }
}
```

이제 이전 소스코드를 옮기는 일은 쉽다고 생각될 것이다. 여기서 한 단계 더 나아가자면 기능별로 파일을 분리할 수 있다.

리스트 기능을 관리하는 useUserRepositories를 만들어보자.

```javascript
// src/composables/useUserRepositories.js

import { fetchUserRepositories } from "@/api/repositories";
import { ref, onMounted, watch, toRefs } from "vue";

export default function useUserRepositories(user) {
  const repositories = ref([]);
  const getUserRepositories = async () => {
    repositories.value = await fetchUserRepositories(user.value);
  };

  onMounted(getUserRepositories);
  watch(user, getUserRepositories);

  return {
    repositories,
    getUserRepositories,
  };
}
```

그리고 검색기능 useRepositoryNameSearch

```javascript
// src/composables/useRepositoryNameSearch.js

import { ref, onMounted, watch, toRefs } from "vue";

export default function useRepositoryNameSearch(repositories) {
  const searchQuery = ref("");
  const repositoriesMatchingSearchQuery = computed(() => {
    return repositories.value.filter((repository) => {
      return repository.name.includes(searchQuery.value);
    });
  });

  return {
    searchQuery,
    repositoriesMatchingSearchQuery,
  };
}
```

이제 두 가지의 파일을 불러와서 사용하면 된다.

```javascript
// src/components/UserRepositories.vue
import useUserRepositories from '@/composables/useUserRepositories'
import useRepositoryNameSearch from '@/composables/useRepositoryNameSearch'
import { toRefs } from 'vue'

setup (props) {
    const { user } = toRefs(props)

    const { repositories, getUserRepositories } = useUserRepositories(user)

    const {
      searchQuery,
      repositoriesMatchingSearchQuery
    } = useRepositoryNameSearch(repositories)

    return {
      repositories: repositoriesMatchingSearchQuery,
      getUserRepositories,
      searchQuery,
    }
```

자 보이시나? 소스코드가 엄청 간결해지고 기능별로 구분되어 있어 코드 분석이 더욱 쉬워질 것이다.

마지막으로 필터기능을 추가하면 전체적인 소스코드는 아래와 같이 된다.

```javascript
// src/components/UserRepositories.vue
import { toRefs } from "vue";
import useUserRepositories from "@/composables/useUserRepositories";
import useRepositoryNameSearch from "@/composables/useRepositoryNameSearch";
import useRepositoryFilters from "@/composables/useRepositoryFilters";

setup(props) {
  const { user } = toRefs(props);

  // 리스트
  const { repositories, getUserRepositories } = useUserRepositories(user);

  // 검색
  const { searchQuery, repositoriesMatchingSearchQuery } =
    useRepositoryNameSearch(repositories);

  // 필터
  const { filters, updateFilters, filteredRepositories } =
    useRepositoryFilters(repositoriesMatchingSearchQuery);

  return {
    repositories: filteredRepositories,
    getUserRepositories,
    searchQuery,
    filters,
    updateFilters,
  };
}
```

# 마무리

Composition API로 작성된 소스코드를 보다가 Vue2의 Options API로 작성된 소스코드를 보면 머리가 아프다. 그만큼 Composition API가 가독성이 너무 좋다. Composition API 작성법만 익숙해지면 행복한 Vue3 라이프가 될 것이다.

# 참조

- Vue3 공식 Document: [https://v3.vuejs.org](https://v3.vuejs.org)

# 프로젝트

개발환경 

python 3.9

django 4.

- requirements.txt

```
Running setup.py install for dj-rest-auth ... done
Running setup.py install for django-seed ... done
Running setup.py install for django-allauth ... done
Running setup.py install for jupyter-latex-envs ... done
Running setup.py install for jupyter-nbextensions-configurator ... done
```



vue3

- router
  - `npm add vue-router@next`이거 안해줘서 며칠을 날린 것 같다...

- vuex
- axios
  - https://amanokaze.github.io/blog/Vuejs-async-await/ - 로그아웃 문제가 있는 줄 알았는데 어찌 해결됨, 오타였던듯..


# 개발 순서

## 회원가입

### django

book/settings.py

```python
# INSTALLED_APPS에 아래 추가

'rest_framework',
'rest_framework.authtoken',

'dj_rest_auth',
'dj_rest_auth.registration',
'allauth',
'allauth.account',
'allauth.socialaccount',

'django_seed',
'django_extensions',

'corsheaders',

# 하단에 아래 코드 추가
AUTH_USER_MODEL = 'accounts.User'

CORS_ALLOW_ALL_ORIGINS = True

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.TokenAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
    'rest_framework.permissions.AllowAny'
    ],
}
```



books/models.py

```python
from django.db import models
from django.contrib.auth.models import AbstractUser


# Create your models here.
class User(AbstractUser):
    pass
```



### vue

@/api/drf.js

```js
const HOST = 'http://localhost:8000/'

const ACCOUNTS = 'accounts/'
const ARTICLES = 'articles/'
const COMMENTS = 'comments/'
const MOVIES = 'movies/'

export default {
  accounts: {
    login: () => HOST + ACCOUNTS + 'login/',
    logout: () => HOST + ACCOUNTS + 'logout/',
    signup: () => HOST + ACCOUNTS + 'signup/',
    // Token 으로 현재 user 판단
    currentUserInfo: () => HOST + ACCOUNTS + 'user/',
    // username으로 프로필 제공
  }
}
```

vuex 설치하면 store파일이 생기는데 store 안에 modules파일 생성 후 `accounts.js`생성 코드는 아래에 있음

```js

// @/store/index.js 하단에 추가?
export default new Vuex.Store({
  modules: { accounts, movies, articles }
})
// @/store/modules/accounts.js
import router from '@/router'
import axios from 'axios'
import drf from '@/api/drf'

export default {
  // namespaced: true,

  state: {
    token: localStorage.getItem('token') || '' ,
    currentUser: {},
    profile: {},
    authError: null,
    recommendation: {}
  },

  getters: {
    isLoggedIn: state => !!state.token,
    currentUser: state => state.currentUser,
    profile: state => state.profile,
    authError: state => state.authError,
    authHeader: state => ({ Authorization: `Token ${state.token}`}),
    recommendation: state => state.recommendation,
  },

  mutations: {
    SET_TOKEN: (state, token) => state.token = token,
    SET_CURRENT_USER: (state, user) => state.currentUser = user,
    SET_PROFILE: (state, profile) => state.profile = profile,
    SET_AUTH_ERROR: (state, error) => state.authError = error,
    SET_RECOMMENDATION: (state, recommendation) => state.recommendation = recommendation
  },

  actions: {
    saveToken({ commit }, token) {
      commit('SET_TOKEN', token)
      localStorage.setItem('token', token)
    },

    removeToken({ commit }) {
      commit('SET_TOKEN', '')
      localStorage.setItem('token', '')
    },

    login({ commit, dispatch }, credentials) {

      axios({
        url: drf.accounts.login(),
        method: 'post',
        data: credentials
      })
        .then(res => {
          const token = res.data.key
          dispatch('saveToken', token)
          dispatch('fetchCurrentUser')
          router.push({ name: 'home' })
        })
        .catch(err => {
          console.error(err.response.data)
          commit('SET_AUTH_ERROR', err.response.data)
        })
    },

    signup({ commit, dispatch }, credentials) {

     axios({
       url: drf.accounts.signup(),
       method: 'post',
       data: credentials
     })
     .then(res => {
       const token = res.data.key
       dispatch('saveToken', token)
       dispatch('fetchCurrentUser')
       router.push({ name: 'home' })

     })
     .catch(err => {
        console.error(err.response.data)
        commit('SET_AUTH_ERROR', err.response.data)
     })
    },

    logout({ getters, dispatch }) {

     axios({
       url: drf.accounts.logout(),
       method: 'post',
       headers: getters.authHeader,
     })
     .then(() => {
      dispatch('removeToken')
      alert('로그아웃 됐습니다.')
      router.push({ name: 'home' })
     })
     .catch(err => {
       console.log(err.response)
     })
    },

    fetchCurrentUser({ commit, getters, dispatch }) {
      
      if (getters.isLoggedIn) {
        axios({
          url: drf.accounts.currentUserInfo(),
          method: 'get',
          headers: getters.authHeader,
        })
          .then(res => commit('SET_CURRENT_USER', res.data))
          .catch(err => {
            if (err.response.status === 401) {
              dispatch('removeToken')
              router.push({ name: 'login' })
            }
          })
      }
    },
    fetchProfile({ commit, getters, dispatch }, username) {
      if (getters.isLoggedIn) {
        axios({
          url: drf.accounts.profile(username),
          method: 'get',
          headers: getters.authHeader,
        })
          .then(res => {
            commit('SET_PROFILE', res.data)
          }
            )
          .catch(err => {
            if (err.response.status === 401) {
              dispatch('removeToken')
              router.push({ name: 'login' })
            }
          })
      }
    },
    fetchRecommendForm({ commit, getters }) {

      axios({
        url: drf.accounts.recommend(),
        method: 'get',
        headers: getters.authHeader,
      })
      .then(res => {
        commit('SET_RECOMMENDATION', res.data)
      }
        )
      .catch(err => {
        console.log(err.response)
      })
    },
    likeGenres({ commit, getters }, recommendationPk ) {
      axios({
        url: drf.accounts.likeRecommend(recommendationPk),
        method: 'post',
        headers: getters.authHeader,
      })
      .then(res => commit('SET_RECOMMENDATION', res.data))
      .catch(err => console.error(err.response))
    },
  }
}
```



## 로그인

### 오류

`Uncaught (in promise) TypeError: Cannot read properties of undefine` - 오류는 뜨는데 작동은 한다....., 오타일 확률이 크다.

## 추천 알고리즘

## 채팅(토론) 구현

django에서 모델로 만들어서 저장하고 불러오고 요청해서 가져옴

- 채팅 보내기
- 삭제하기 
- 채팅 내역 가져오기

- 누구와 채팅 했는지도 알아야함
- 사용자 검색해서 채팅 보낼 수 있어야함

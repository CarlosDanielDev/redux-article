# Redux

![data-flux](https://i.imgur.com/cE9PWTI.png)


## Exemplo

A implementação do **Redux** segue no pattern de `ducks`, com uma pasta **store**, e uma pasta com os **ducks** (os estados da aplicação), 
dentro de cada _duck_, tem um padrão de arquivos, são eles:
- actions
- sagas
- types
- index(reducer)

### Types
Está é uma camada que poderia se considerar opcional, pois efetivamente ela não influencia e nem faz parte do fluxo de dados, mas ela se faz muito necessário pois estamos usando o `typescript` e pra que haja uma organização nos tipos de cada `duck` dentro da **store**. Organizando os tipos podemos aproveitar da vantagem do **IntelliSense** do editor, é nesta camada  onde é especificado o tipo de cada **action** e o tipo de cada **state** dentro da **store**, abaixo estão uns exemplos de interfaces para `Type Actions` e `Type State`.

```typescript
export enum ActionTypes {
  LOAD = '@my_duck/LOAD',
  LOAD_SUCCESS = '@my_duck/LOAD_SUCCESS',
  LOAD_FAILURE = '@my_duck/LOAD_FAILURE',
  ADD = '@my_duck/ADD'
}

export interface MyDuck {
  name: string
  email: string
  phone: number
}

export interface State {
  readonly data: MyDuck[]
  readonly loading: boolean
  readonly error: boolean
}
```

### Actions
As **actions** são as assinaturas dos métodos, ou seja o formato como serão disparadas na camada de **UI**,
ou seja uma `action` é ativada por uma chamada na **UI**, exemplo de uso com e sem parâmetros:
```typescript
const dispatch = useDispatch()

// WITH PARAMS (payload)
dispatch({
  type: 'type_of_my_action',
  payload: {
    ...data_of_my_payload
  }
})

//WITHOUT PARAMS (payload)

dispatch({
  type: 'type_of_my_action'
})
```
Abaixo aqui está a assinatura de algumas **actions**, com e sem parâmetros.
```typescript
// Actions Without Params

export const load = () => {
  return {
    type: ActionTypes.LOAD
  }
}

export const loadFailure = () => {
  return {
    type: ActionTypes.LOAD_FAILURWE,
  }
}

// Action With params

export const loadSuccess = (data: TypeOfData) => {
  return {
    type: ActionTypes.LOAD_SUCCESS',
    payload: {
      data
    }
  }
}
```

### Sagas
A camada de `saga`, é onde nós chamamos os serviços externos, ou seja onde nós chamamos as **APIs**, abaixo vou deixar um exemplo de uma chamada com a implementação do saga que usam `generators`:

```typescript
import { call, put } from 'redux-saga/effects'
import { loadFailure, loadSuccess } from './actions'

//Without Params
export function* load() {
  try {
    yield refresh()
    const response = yield call(api.get, '/get-something')
    if(response?.data) yield put(loadSuccess(response?.data))
  } catch {
    yield put(loadFailure())
  }
}

// With Params
export function* add({payload}: any) {
  try {
    const request = yield call(api.post, '/post-something', {
     data: [{ ...payload.data }]
    });
    // do something if success (put actions)
  } catch (error) {
    // do something if not success (put actions)
  }
}
```

### Reducers
O **reducer** é o cara que vai tratar os dados, é importante que cada `state` dentro da **store** tenha seu próprio `reducer`, o **reducer** executa a modificação nos dados que e refletida em toda a UI que consome o respectivo **state** da _**store**_, cada reducer recebe 2 parâmetros, o state inicial e a action, a action por sua vez é como o objeto retornado das actions que ja foram criada: `{ type: 'type', payload: {data} }`, aqui abaixo segue um exemplo de um `reducer` na prática.
```typescript
import { Reducer } from 'redux'
import { State, ActionTypes } from './types'

const INITIAL_STATE: State = {
  data: [],
  loading: false,
  error: false
}

export const reducer: Reducer<State> = (state = INITIAL_STATE, {payload, type}) => {
  switch(type) {
    case ActionTypes.LOAD:
      return {
        ...state,
        loading: true
      }
    case ActionTypes.LOAD_SUCCESS:
      return {
        ...state,
        loading: false,
        data: [...payload.data]
      }
    case ActionTypes.FAILURE:
      return {
        ...state,
        error: true,
        loading: false
      }
    case ActionTypes.ADD:
      return {
        ...state
      }
  }
}
```

Beleza, agora temos nossos types, actions, sagas e reducers criados, agora como tudo isso se conecta ? ai vem a mágica da arquitetura flux, em nossa pasta store temos o seguinte esquema de pastas:
```
store
├── ducks
│   ├── my_duck
│   │   ├── actions.ts
│   │   ├── index.ts
│   │   ├── sagas.ts
│   │   └── types.ts
│   ├── rootReducer.ts
│   └── rootSaga.ts
└── index.ts
```

A pasta _**ducks**_ guarda todos os nossos **states**. Agora temos 3 tópicos pra abordar, o **rootSaga**, **rootReducer** e o _index_ da nossa **store**.
- Root Saga
- Root Reducer
- Store

### Root Saga

O arquivo `rootSaga.ts` é onde conectamos todos os sagas da nossa arquitetura, é o arquivo onde eu vou importar todos os nosso métodos que lidam com serviços externos, aqui abaixo vou deixar um exemplo com a implementação conectando o `saga` que nós criamos.
```typescript
import { all, takeLatest } from 'redux-saga/effects'

import { ActionTypes } from './my_duck/types'

import { load, add } from './my_duck/sagas'

export default function* rootSaga() {
  return yield all([
    takeLatest(ActionTypes.LOAD, load),
    takeLatest(ActionTypes.ADD, add),
  ])
}

``` 
E pronto, assim nos conectamos todos os nosso sagas.

### Root Reducer
O arquivo `rootReducer.ts` é o arquivo que faz o combine de todos os nosso **reducers**, ele faz que tudo seja refletido ao mesmo tempo na nossa **store**, ele pega todos os nossos _reducers_ e basicamente cria **listeners** que observam o **dispatch** e executa o respectivo **reducer** de acordo com a **action** executada no **dispatch**, ex:
```typescript
import { combineReducers } from 'redux'
import { reducer as my_duck } from './my_duck'

export default combineReducers({
  my_duck,
})

```
é um arquivo bem simples onde combinamos nossos **reducers**.

### Store (index)
Aqui é onde a mágica acontece, onde a gente junta tudo isso que foi criado e faz funcionar, nesse arquivo nós criamos o **store** da aplicação, implementamos o **rootSaga** e o **rootReducer**, abaixo segue o exemplo de implementação do **store**:
```typescript
import { createStore, Store, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'
import rootReducer from './ducks/rootReducer'
import rootSaga from './ducks/rootSaga'
import { State } from './ducks/my_duck/types'

export interface AppState {
  my_duck: State
}
// Create Sagas Middleware
const sagaMiddleware = createSagaMiddleware()
// create store variable and apply reducers and saga middleware
const store: Store<AppState> = createStore(rootReducer, applyMiddleware(sagaMiddleware))

sagaMiddleware.run(rootSaga)

export default store
```

Pronto, temos nosso arquitetura de redux finalizada.


### Uso do Store
Exportamos nossa variável de store, Agora precisamos criar o _**provider**_ por volta da Aplicação, então no index de toda a aplicação fica dessa forma:
```typescript
import React from 'react'
import Routes from './routes'
import { Provider } from 'react-redux'
import store from './store'

export default const App = () => {
  return (
    <Provider store={store}>
      <Routes/>
    </Provider>
  )
}
```

Para utilizar um _**state**_ da nossa **store**, é bem simples basta importa o **hook**  **`useSelector`** e acessar o estado desejado, segue um exemplo abaixo.
```typescript
import { AppState } from './store'
...
const data = useSelector((state: AppState) => state.my_duck)

// Pronto agora dentro dessa constante **data** temos todos os dados do nosso state.
```

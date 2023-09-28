# RFC

- Data de início: 30-08-2022
- RFC PR: 

# Sumário

Essa proposta de feature inclui dois pontos, o primeiro é uma nova forma de
gerenciamento de estados globais com o [Zustand](https://github.com/pmndrs/zustand),
o segundo ponto é alterar a forma de persistir os dados, substituir o
[AsyncStorage](https://github.com/react-native-async-storage/async-storage/),
pelo [MMKV](https://github.com/mrousavy/react-native-mmkv/).

Hoje temos alguns estados globais no aplicativo, e estão aparecendo cada vez mais,
sempre com o mesmo proposito de salvar pequenas informações de dados para se 
utilizar em algum fluxo, o Context funciona muito bem, mas começamos a ter muitos
Providers e deixar a aplicação cada vez mais complexa com tantos estados. O Zustand
resolve vários dos pontos, pois não é necessário adicionar um Provider na aplicação,
além disso funciona perfeitamente para criação de pequenos estados globais para
usos específicos.

Sobre a persistência de dados o AsyncStorage funciona muito bem, mas por ser async
temos alguns problemas de race conditions, principalmente quando iniciamos o 
aplicativo, além de necessitar de Promise para persistir valores. O MMKV é um
storage síncrono e absurdamente performático criado pela Tencent para ser utilizado
no Wechat, nesse caso temos a implementação para React Native. Além de ser síncrono
vem com uma API completa para armazenar todos os tipos de dados e integrar com o
Zustand.

# Exemplo básico

Um exemplo básico retirado da PR criada como poc linkada abaixo:

```tsx
interface UserLoginInformation {
  email?: string
  name?: string
  provider?: string
}

interface LoginInformationState {
  loginInformation: UserLoginInformation
  setLoginInformation: (loginInformation: UserLoginInformation) => void
}

export const useLoginInformation = create<LoginInformationState>()(
  persist(
    set => ({
      loginInformation: {},
      setLoginInformation: loginInformation => set({ loginInformation }),
    }),
    {
      name: StorageKeys.userLoginInformation,
      storage: createJSONStorage(() => zustandStorage),
    }
  )
)
```

Nesse exemplo podemos ver que a tipagem funciona perfeitamente com a função
`create` do Zustand, com isso podemos utilizar um storage customizado, que nesse
caso utiliza o storage do MMKV.

A implementação utilizada acima do MMKV nada mais é que uma formatação para ficar
com a mesma API de métodos do LocalStorage do browser:

```ts
export const storage = new MMKV()

export const zustandStorage: StateStorage = {
  setItem: (name, value) => {
    return storage.set(name, value)
  },
  getItem: name => {
    const value = storage.getString(name)
    return value ?? null
  },
  removeItem: name => {
    return storage.delete(name)
  },
}
```

Para ver mais detalhes da implementação verificar a 
[Pull Request da poc]().

# Motivação

A algum tempo temos problema com a atual AppStore que surge em diversos momentos,
além de utilizar um MobX muito antigo, não tendo reatividade para componentes
funcionais. Com isso conseguimos criar pequenos Context's com certas informações
que precisamos compartilhar entre telas, mas com o aumento desses Context's a
aplicação começa a ficar mais complexa, com muita lógica nesses contextos e com
o tempo perdendo performance, pois não utilizamos nenhum selector para esses
contextos.

O AsyncStorage é bastante funcional e está na comunidade React Native a um tempo,
mas por ser assíncrono ainda existem alguns problemas de race condition entre
promises, tratamentos que devem ser feitos por ser assíncrono e alguns
workarounds esperando o valor usando custom hook. Com o MMKV temos um storage
muito mais rápido, síncrono, usando a nova arquitetura do React Native e facilitando
muito o uso.

# **Design** detalhado

A ideia inicial dessa proposta surgiu para remover a classe AppStore do nosso
aplicativo, mas além disso se somou a necessidade de diminuir a quantidade de
contextos que estão surgindo e unificar uma forma de compartilhar estados pela
aplicação.

Na integração inicial já será disponibilizada toda uma API para apenas utilizar,
ou seja, o Zustand e o MMKV estarão disponíveis e configurados no aplicativo.

A ideia de arquitetura para as stores do Zustand é separá-las em pastas dentro
do mesmo contexto que os dados, por exemplo, o contexto de `User` do nosso aplicativo
teria stores para armazenar as informações de login, endereço, etc...

Então no lugar das pastas `contexts` teríamos a pasta `stores` com os hooks criados
pelo Zustand, com isso conseguimos criar pequenos estados globais para serem
utilizado onde precisarmos.

Na parte de testes, já terá um arquivo de mock do zustand que é usado para simular
uma store na hora de rodar nossos testes com o jest.

Outro ponto é que mesmo o Zustand gerando hooks com o método `create`, podemos
utilizar métodos internos desses hooks, então podemos pegar dados e adicionar
dados fora de componentes React, como por exemplo:

```js
const useDogStore = create(() => ({ paw: true, snout: true, fur: true }))

// Getting non-reactive fresh state
const paw = useDogStore.getState().paw
// Listening to all changes, fires synchronously on every change
const unsub1 = useDogStore.subscribe(console.log)
// Updating state, will trigger listeners
useDogStore.setState({ paw: false })
// Unsubscribe listeners
unsub1()

// You can of course use the hook as you always would
const Component = () => {
  const paw = useDogStore((state) => state.paw)
  ...
```

Nesse exemplo ainda é criado um `subscribe` para observar as alterações da store,
outro ponto é que essa forma de utilizar facilitar MUITO em nossos testes, como:

```js
it('display saved user email', () => {
  const email = 'johndoe@email.com'
  useLoginInformation.setState({ loginInformation: { email } })

  const { getByText } = renderWithMocks([])(<Welcome {...props} />)

  expect(getByText(email)).toBeTruthy()
})
```

# Desvantagens

Algumas desvantagens que enxergo no momento:

- curva de aprendizado com as ferramentas, mesmo sendo pequeno sempre existe
- tempo para refatorar os contextos já criados
- alterar locais que estão utilizando o AsyncStorage
- cuidado ao testar locais que foram refatorados
- a AppStore hoje possui muita lógica, tomar muito cuidado com as migrações
- a react-native-mmkv tem código nativo, atualizar sempre que possível

# Alternativas

Temos outras ferramentas para gerenciamento de estado como Redux, Jotai, atualizar
o MobX, mas hoje enxergo o Zustand como a melhor opção, pois é uma biblioteca apenas
com os middlewares necessários internamente, é pequena tanto no tamanho de bundle,
quanto na forma de se utilizar, além de ser robusta quando é preciso criar lógicas
complexas como selectors com várias camadas.

Sobre a forma de persistir dados, uma das opções é manter o AsyncStorage. Com isso
não teríamos o esforço de migrar para outro storage, mas ao mesmo tempo ainda 
teríamos o problema de ser assíncrono e "lento" se comparada as outras opções.

# Estratégia de adoção

A estratégia inicial seria deixar todas as pessoas dos squads por dentro das
ferramentas e como utilizá-las, depois do aceite desse RFC começaríamos a refatorar
alguns lugares que estão utilizando o Context e o AsyncStorage, e com isso criar
uma documentação sobre como fazer isso.

Outra parte da estratégia seria eliminar os dados da AppStore aos poucos, para
conseguirmos testar cada parte da refatoração, assim eliminando essa classe.

# Tutoriais e exemplos

Sobre o Zustand eu até poderia deixar algum exemplo básico, mas o readme do repositório
é uma documentação completa, com mais diversos exemplos de implementações e 
explicações bem melhores do que eu conseguiria fazer:
https://github.com/pmndrs/zustand

O react-native-mmkv é muito simples de implementar e utilizar, podemos utilizar
o storage, ou dentro de componentes podemos usar hooks:

```ts
import { MMKV } from 'react-native-mmkv'

export const storage = new MMKV()

storage.set('user.name', 'Marc')
storage.set('user.age', 21)
storage.set('is-mmkv-fast-asf', true)

const username = storage.getString('user.name') // 'Marc'
const age = storage.getNumber('user.age') // 21
const isMmkvFastAsf = storage.getBoolean('is-mmkv-fast-asf') // true

// checking if a specific key exists
const hasUsername = storage.contains('user.name')

// getting all keys
const keys = storage.getAllKeys() // ['user.name', 'user.age', 'is-mmkv-fast-asf']

// delete a specific key + value
storage.delete('user.name')

// delete all keys
storage.clearAll()

const user = {
  username: 'Marc',
  age: 21
}

// Serialize the object into a JSON string
storage.set('user', JSON.stringify(user))

// Deserialize the JSON string into an object
const jsonUser = storage.getString('user') // { 'username': 'Marc', 'age': 21 }
const userObject = JSON.parse(jsonUser)
```

```tsx
function App() {
  const [username, setUsername] = useMMKVString("user.name")
  const [age, setAge] = useMMKVNumber("user.age")
  const [isPremiumUser, setIsPremiumUser] = useMMKVBoolean("user.isPremium")
}
```

Mais no repositório: https://github.com/mrousavy/react-native-mmkv/

Mas é claro que esses casos são muito específicos, pois essa proposta é utilizar
o MMKV juntamente com o Zustand, então são dois pontos diferentes que são atacados,
o Zustand é responsável por armazenar o estado local e ser reativo dentro do React,
enquanto o MMKV é responsável por persistir esses dados. Só que, quando utilizamos
a integração do MMKV dentro do middleware de persist do Zustand, praticamente não
vamos manipular os estados da forma padrão do MMKV, somente quando precisarmos de
estados locais que necessitam ser persistidos, ai não precisamos do Zustand.

Integração: https://github.com/mrousavy/react-native-mmkv/blob/master/docs/WRAPPER_ZUSTAND_PERSIST_MIDDLEWARE.md

# Documentação

<br/>
<div align="center">

<br/>

# Gerenciamento e persistência de estados com Zustand e MMKV

### Como e quando utilizar o Zustand para gerenciar um estado e MMKV para persistir esse dado.

<br/>

</div>

***

### Motivação

Quando falando de gerenciamento de estados do aplicativo logo pensamos na [AppStore](), essa classe existe a aproximadamente 4 anos, em uma época que o MobX ganhou muito espaço no React com componentes de classe, fornecendo decorators para gerenciar os estados e criar actions, reactions, etc. O problema é que essa estrutura não fornece reatividade para componentes funcionais que usamos com hooks e a classe está grande e complexa para fazer uma migração total, então começamos a usar a Context API pontualmente.

Com isso, hoje temos alguns estados globais no aplicativo, e estão aparecendo cada vez mais, sempre com o mesmo propósito de salvar pequenas informações de dados para se utilizar em algum fluxo, o Context funciona muito bem, mas começamos a ter muitos Providers e deixar a aplicação cada vez mais complexa com tantos estados. O [Zustand](https://github.com/pmndrs/zustand) resolve vários dos pontos, pois não é necessário adicionar um Provider na aplicação, além disso funciona perfeitamente para criação de pequenos estados globais para usos específicos.

Sobre a persistência de dados o AsyncStorage funciona muito bem, mas por ser async temos alguns problemas de race conditions, principalmente quando iniciamos o aplicativo, além de necessitar de Promise para persistir valores. O [MMKV](https://github.com/Tencent/MMKV) é um storage síncrono e absurdamente performático criado pela Tencent para ser utilizado no Wechat, nesse caso temos a implementação para [React Native](https://github.com/mrousavy/react-native-mmkv). Além de ser síncrono vem com uma API completa para armazenar todos os tipos de dados e integrar com o Zustand.

Para entender um pouco mais da ideia, temos uma [RFC]() sobre a escolha das duas ferramentas.

### Como utilizar

#### Tenho um estado global sem persistência

Em alguns casos precisamos de um estado global para a aplicação que não precisam ser persistidos a cada reinicialização do App, com isso podemos criar uma store do Zustand simples e só utilizarmos nos locais que precisamos desses dados. Então, o Zustand consegue resolver o problema sem a necessidade do Context, MMKV e nenhum provider.

Temos um exemplo no contexto da [Splash Screen]()

Para facilitar fica um exemplo de um contador que adiciona e subtrai 1 do estado global com duas funções:

```ts
import { create } from 'zustand'

interface CounterState {
  count: number
  add: () => void
  sub: () => void
}

export const useCounter = create<CounterState>()(set => ({
  counter: 0,
  add: () => {
    set(state => ({ counter: state.counter + 1 }))
  },
  sub: () => {
    set(state => ({ counter: state.counter - 1 }))
  },
}))
```

Com isso podemos utilizar em componentes e utilizar em funções de forma programática:

```tsx
export const Counter = () => {
  const counter = useCounter(state => state.counter)

  return (
    <View>
      <Text>{counter}</Text>
    </View>
  )
}
```

E em funções, mantendo a reatividade:

```ts
export const someExternalFunction = () => {
  // ...some logic
  useCounter.setState(state => ({ counter: state.counter + 1 }))
  useCounter.getState().add()
  useCounter.getState().sub()
  // ...some logic
}
```

#### Tenho um estado local que precisa ser persistido

Nem todo estado que necessita ser persistido precisa ser global, as vezes um componente tem um estado local mas que precisa ser salvo entre as reinicializações do app, nesse caso é bem fácil, a lib react-native-mmkv já nos fornece uma [gama de hooks](https://github.com/mrousavy/react-native-mmkv/blob/master/docs/HOOKS.md) que podemos utilizar em nossos componentes.

Temos um exemplo em um hook de [Deeplink]()
Observação: Nesse fluxo é setado um valor quando o app é inicializado e por isso conseguimos recuperar nesse hook, mas vale lembrar que **não temos reatividade** caso o valor seja alterado em outro componente utilizando o hook (nesse caso usar o Zustand + MMKV) e nem quando é alterado de forma programática usando a store do mmkv, nesse caso checar o [hook de listener](https://github.com/mrousavy/react-native-mmkv/blob/master/docs/HOOKS.md#listen-to-value-changes).

Exemplo de um componente com um contador local mas que precisa ser persistido:

```tsx
export const Counter = () => {
  const [counter, setCounter] = useMMKVNumber('counter') // Utilizar StorageKeys

  const add = () => {
    setCounter(state => state + 1)
  }

  const sub = () => {
    setCounter(state => state + 1)
  }

  return (
    <View>
      <TouchableOpacity >Add</TouchableOpacity>
      <Text>{counter}</Text>
      <TouchableOpacity >Sub</TouchableOpacity>
    </View>
  )
}
```

#### Tenho um estado global que precisa ser persistido

Bom, esse é um caso clássico de quando iríamos utilizar Context + Async Storage, mas aqui conseguimos juntar o melhor dos dois mundos usando Zustand e o MMKV Storage.

Para essa integração ser feita foram criados alguns arquivos de setup, entre eles a [storage em si do MMKV](https://github.com/mrousavy/react-native-mmkv/blob/master/src/services/storage/MMKVStorage.ts) e o [zustandStorage](https://github.com/mrousavy/react-native-mmkv/blob/master/src/services/storage/zustand/zustandStorage.ts), com eles criados só precisamos criar uma store do Zustand parecida com a do primeiro exemplo.

Um exemplo no projeto criado utilizando essa abordagem para exibir a [badge na Tab Bar]()

Então pensando no setup que já temos no projeto, podemos criar um exemplo de contador global persistido da seguinte forma:

```ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

import { StorageKeys } from 'services/storage/StorageKeys'
import { zustandStorage } from 'services/storage/zustand/zustandStorage'

interface CountState {
  count: number
  add: () => void
  sub: () => void
}

export const useCount = create<CountState>()(
  persist(
    set => ({
      count: 0,
      add: () => {
        set(state => ({ count: state.count + 1 }))
      },
      sub: () => {
        set(state => ({ count: state.count - 1 }))
      },
    }),
    {
      name: StorageKeys.counter,
      storage: createJSONStorage(() => zustandStorage),
    }
  )
)
```

E como no primeiro exemplo podemos utilizar dá mesma forma e o Zustand fica responsável por toda a persistência no MMKV com o middleware persist utilizado no setup da store.

```tsx
import { shallow } from 'zustand/shallow'

export const Counter = () => {
  const { counter, add } = useCounter(state => ({ counter: state.counter, add: state.add }), shallow)

  return (
    <View>
      <Text>{counter}</Text>
      <TouchableOpacity >Sub</TouchableOpacity>
    </View>
  )
}
```

Acima utilizamos o shallow, pois estamos fazendo a seleção de mais de uma parte do estado e para renderizar na hora correta é utilizado o shallow para comparação dos dados recebidos.

```ts
export const someExternalFunction = () => {
  // ...some logic
  useCounter.setState(state => ({ counter: state.counter + 1 }))
  useCounter.getState().add()
  useCounter.getState().sub()
  // ...some logic
}
```

## Estrutura

A estrutura foi idealizada inicialmente na [RFC](), mas aqui podemos detalhar melhor a ideia de como utilizar as stores.

Como criamos pequenos estados globais, a estrutura definida foi criar uma pasta `stores` dentro do contexto que o dado da store se refere, por exemplo `Order`. Então uma store com o contexto de pedido fica na seguinte estrutura.

```
src/
├─ scenes/
│  ├─ Order/
│  │  ├─ stores/
│  │  │  ├─ useOrderRatingBadge.ts
```

## Migração

A ideia é com o tempo migrarmos os dados da AppStore e os dados que estão utilizando a Context API para utilizar essa nova estrutura, mas é algo a ser feito aos poucos e garantindo que tudo se mantém funcionando, se for algo sendo criado do zero vai ser mais simples adotar os padrões citados acima e seguir as documentações das ferramentas para algo mais complexo.

Caso seja necessário migrar dados do AsyncStorage para o MMKV, a documentação [fornece uma função](https://github.com/mrousavy/react-native-mmkv/blob/master/docs/MIGRATE_FROM_ASYNC_STORAGE.md) para copiarmos todos os dados para o MMKV.

Como a migração será feita em partes, também podemos criar funções menores baseadas na da documentação e removermos em uma próxima Pull Request, vai de cada pessoa e feature, acredito que na maioria dos casos o dado possa ser perdido e recriado com a implementação do MMKV, já que os dados já são resetados a cada reinstalação do app.

## Referências

O intuito dessa documentação é passar um panorama geral de como funcionam as ferramentas e como utilizar para cada necessidade, mas não caberia todas as features que as ferramentas oferecem nessa documentação, também vão existir casos que precisarão de alguma feature mais complexa, alguma implementação, middleware e outras coisas muito específicas, por isso deixo o link da documentação das duas ferramentas para servir de base para implementações:

- [zustand](https://github.com/pmndrs/zustand)
- [react-native-mmkv](https://github.com/mrousavy/react-native-mmkv)

# JobSource Work Management Platform Frontend

[![Build Status](https://dev.azure.com/KraferdSoftware/wmp/_apis/build/status/card-ui?branchName=main)](https://dev.azure.com/KraferdSoftware/wmp/_build/latest?definitionId=1&branchName=main)

This is the Job Source frontend application.

## Getting Started

1. You'll need [NodeJS](https://nodejs.org/en/).
2. Run `npm install` then `npm run start:dev` to start the mock server and frontend application. Navigate to `http://localhost:4000`.
3. Check below for app architecture. Many Redux patterns used can be found in this [gothinkster app](https://github.com/gothinkster/react-redux-realworld-example-app) so you may find their docs helpful as well.

## Contribute

TODO: Explain how other users and developers can contribute to make your code better.

## React-Redux App Architecture

### Store Provider and Routes

File: `App.ts`

Renders routes pointing to their associated components:

```tsx
const App = () => {
	return (
		<Provider store='{store}'>
			<ConnectedRouter history='{history}'>
				<Switch>
					<Route exact path='/' component='{Home}' />
					<Route path='/signin' component='{Signin}' />
					<Route path='/registration' component='{Registration}' />
					<Route path='/dashboard' component='{Dashboard}' />
					<Route path='/password-reset' component='{PasswordReset}' />
				</Switch>
			</ConnectedRouter>
		</Provider>
	);
};
```

### Agent

File: agent.ts

Exports an object where each key is a "service" and a service has methods that internally run a request:

- `get`
- `put`
- `post`
- `delete`

For example, `Auth`:

```typescript
const Auth = {
	current: () => {
		return requests.get('/user');
	},
	register: (email: string, password: string) => {
		return requests.post('/users', {email, password});
	},
	reset: ({password, token}: {password: string; token: string | null}) => {
		return requests.put('/user/reset-password', {password, token});
	},
	signin: (email: string, password: string) => {
		return requests.post('/users/signin', {email, password});
	},
	signout: (userId: number) => {
		return requests.post('/users/signout', {userId});
	},
};
```

These services take options, map to a request, and return a promise. The general type could be:

```typescript
type Service = {
	[key: string]: (opts: any) => Promise<T>;
};
```

As well, `agent.ts` locally stores a token which can be set via the exported `setToken`. As some config there is `API_ROOT`.

### Middleware

File: `middleware.ts`

Imports: `agent.ts`

#### promiseMiddleware

Intercepts all actions where `action.payload` is a Promise. In which case it:

1. `store.dispatch({ type: 'ASYNC_START', subtype: action.type })`
2. `action.payload.then`
   - success: sets `action.success = true` and does `store.dispatch({ type: 'ASYNC_END', promise: res })`
   - error: sets `action.error = true` and does `store.dispatch({ type: 'ASYNC_END', promise: errRes.response.body })`
3. Then, for success and error, using the modified `action` object: `store.dispatch(action)`

#### localStorageMiddleware

Runs after promiseMiddleware. Intercepts `REGISTRATION | SIGNIN` and either

- a. sets jwt token into localstorage and `agent.setToken(token)`
- b. sets jwt token in localstorage to `''` and does `agent.setToken(null)`

### Reducers

File: `reducer.ts`

Imports: `./reducers/*.ts`

Uses `combineReducers` to export a reducer where each key is the reducer of the file with the same key.

#### General Reducer Patterns

- map payload into piece of state
- toggle loading states by casing on ASYNC_START and action.subtype

```typescript
case 'ASYNC_START':
  if (action.subtype === 'SIGNIN' || action.subtype === 'REGISTRATION') {
    return { ...state, inProgress: true };
  }
```

- toggle errors by taking `action.errors` if it is there (see middleware)

```typescript
case 'REGISTRATION':
  return {
    ...state,
    inProgress: false,
    errors: action.error ? action.payload.errors : null
  };
```

- set state keys to null if payload does not have errors

```typescript
 { errors: action.error ? action.payload.errors : null, }
```

- handle redirections (will be triggered by `useEffect` hooks in `App.tsx`)

```typescript
case 'REDIRECT':
  return { ...state, redirectTo: null };
case 'SIGNOUT':
  return { ...state, redirectTo: '/', token: null, currentUser: null };
```

### UI Components

For styling the application utilizes [Styled-Components](https://styled-components.com/). Styled-Components lets you write actual CSS code to style your components. Pass props into the styled-component element so you can access the props.

Example:

```tsx
const Checkbox = styled.input<{isChecked: boolean}>`
	background-color: ${(props) => (props.isChecked ? 'blue' : 'red')};
`;

const Component = ({isChecked}) => {
	return <Checkbox isChecked={isChecked} />;
};
```

If you have VSCode a helpful extension to have would be the [vscode-styled-components extension](https://marketplace.visualstudio.com/items?itemName=jpoissonnier.vscode-styled-components). This adds syntax highlighting and autocomplete directly in your JS or TS files.

### Dev Server

`dev_server` folder has an express server to handle HTTP requests. This server is not meant to be used for production purposes. Instead it acts as a very generic server from which a frontend application can make requests to. When running builds this folder should be excluded. `API_ROOT` variable on the frontend should be changed accordingly:

```typescript
const API_ROOT = process.env.NODE_ENV === 'development' ? 'http://localhost:8000' : 'PROD_ENDPOINT_HERE'; // prettier-ignore
```

#### Environmental Variables

There's a `.env.sample` file in the root directory. Make a copy and name it `.env`. You only need `APP_SECRET` to get the development server running. `APP_SECRET` is used for signing cookies. The other variables are for an email sandbox service called [Mailtrap](https://mailtrap.io/). If you want to test things like password resets then you'll need an account.

## Build and Test

1. To run tests `npm run test`.
2. If you make any changes to component styling you'll want to update the snapshot tests. Just press 'u' in the terminal.
3. Verify that tests pass before pushing any code.
4. To run build `npm run build`.
# card-ui

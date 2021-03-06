# OpenAPI Client

Generate ES6 or Typescript service integration code from an OpenAPI 2.0 spec.

Also supports optional Redux action creator generation.

Tested against JSON services.

## Install

In your project

    npm install openapi-client --save-dev

Or globally to run CLI from anywhere

    npm install openapi-client -g

### Type Dependencies

If targeting TypeScript you'll also need to install the `isomorphic-fetch` typings in your project.

    npm install @types/isomorphic-fetch --save-dev

## Usage – Generating the API client

`openapi-client` generates action creators in the `outDir` of your choosing. The rest of the examples assume that you've set `--outDir api-client`. You can generate the `api-client` either using the CLI, or in code.

### CLI

```
Usage: openapi [options]

Options:

  -h, --help              output usage information
  -V, --version           output the version number
  -s, --src <url|path>    The url or path to the Open API spec file
  -o, --outDir <dir>      The path to the directory where files should be generated
  -l, --language <js|ts>  The language of code to generate
  --redux                 True if wanting to generate redux action creators
```

### Code

```javascript
const openapi = require('openapi-client')
openapi.genCode({
  src: 'http://petstore.swagger.io/v2/swagger.json',
  outDir: './src/service',
  language: 'ts',
  redux: true
})
.then(complete, error)

function complete(spec) {
  console.info('Service generation complete')
}

function error(e) {
  console.error(e.toString())
}
```

## Usage – Integrating into your project

### Initialisation

If you don't need authorization, or to override anything provided by your OpenAPI spec, you can use the actions generated by `openapi-client` directly. However, most of the time you'll need to perform some authorization to use your API. If that's the case, you can initialize the client, probably in the `index.js` of your client-side app:

```javascript
import serviceGateway from './path/to/service/gateway';

serviceGateway.init({
  url: 'https://service.com/api', // set your service url explicitly. Defaults to the one generated from your OpenAPI spec
  getAuthorization // Add a `getAuthorization` handler for when a request requires auth credentials
});

// The param 'security' represents the security definition in your OpenAPI spec a request is requiring
// For bearer type it has two properties:
// 1. id - the name of the security definition from your OpenAPI spec
// 2. scopes - the token scope(s) required
// Should return a promise
function getAuthorization(security) {
  switch (security.id) {
    case 'account': return getAccountToken(security);
    // case 'api_key': return getApiKey(security); // Or any other securityDefinitions from your OpenAPI spec
    default: throw new Error(`Unknown security type '${security.id}'`)
  }
};

function getAccountToken(security) {
  const token = findAccountToken(security); // A utility function elsewhere in your application that returns a string containing your token – possibly from Redux or localStorage
  if (token) return Promise.resolve({ token: token.value });
  else throw new Error(`Token ${type} ${security.scopes} not available`);
}
```

### Using generated Redux action creators

You can use the generated API client directly. However, if you pass `--redux` or `redux: true` to `openapi-client`, you will have generated Redux action creators to call your API (using a wrapper around `fetch`). The following example assumes that you're using `react-redux` to wrap action creators in `dispatch`.

In your component:

```jsx
import React, { Component } from 'react';
import { connect } from 'react-redux';
import { bindActionCreators } from 'redux';
import functional from 'react-functional';

import { getPetById } from '../api-client/action/pet';

const Pet = ({ actions, pet }) => (
  <div>
    {pet.name}
  </div>
)

// Dispatch an action to get the pet when the component mounts. Here we're using 'react-functional', but this could also be done using the class componentDidMount method
Pet.componentDidMount = ({ actions }) => actions.getPetById(id);

const mapStateToProps = state => (
  {
    pet: getPet(state) // a function that gets 
  }
);

const mapDispatchToProps = dispatch => (
  {
    actions: bindActionCreators({ getPetById }, dispatch)
  }
);

export default connect( mapStateToProps, mapDispatchToProps)(functional(Pet));
```

The client can't generate your reducer for you as it doesn't know how merge the returned object into state, so you'll need to add a something to your reducer, such as:

```jsx
export default function reducer(state = initialState, action) {
  switch (action.type) {
    case GET_PET_BY_ID_START:
      return state.set('isFetching', true);
    case GET_PET_BY_ID: // When we actually have a pet returned
      if(!action.error){
        return state.merge({
          isFetching: false,
          pet: action.payload,
          error: null,
        });
      }
      else{ // handle an error
        return state.merge({
          isFetching: false,
          error: action.error,
        });
      }
    default:
      return state;
  }
}
```

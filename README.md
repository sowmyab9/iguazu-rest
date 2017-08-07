# Iguazu REST
A redux REST caching library that follows Iguazu patterns.

## Usage
### config
Iguazu REST uses a config object that allows you to register resources and provide default and override behavior.



```javascript
import { config } from 'iguazu-rest';

Object.assign(config, {
  // the resources you want cached
  resources: {
    // this key will be used in actions
    users: {
      // returns the url and opts that should be passed to fetch
      fetch: () => ({
        url: `${process.env.HOST_URL}/users/:id`,
        // opts that will be sent on every request for this resource
        opts: {
          credentials: 'include',
        },
      }),
      // optionally override the resources id key, defaults to 'id'
      idKey: 'userId',
    }
  },
  // opts that will be sent along with every resource request
  defaultOpts: {
    headers: {
      "Accept": "application/json",
      "Content-Type": "application/json",
    }
  },
  // extend fetch with some added functionality
  baseFetch: fetchWith6sTimeout,
  // override state location, defaults to state.resources
  getToState: (state) => state.data.resources
});

// Freeze the config if you don't want it to mutate after initialization
Object.freeze(config);
```

#### Resource Fetch Function
In an ideal world, your REST API follows all the [right patterns and best practices](http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api#useful-post-responses). If that is the case your fetch function can return a simple url and iguazu-rest will fill in the parameters based on the id you pass to the action. Not all of us live in an ideal world though and sometimes your boss tells you that you need to support some API endpoints that cannot be made RESTful because it's too much effort. Maybe you need to put the resource ID in the header instead of the URL or you need to make a POST to load a resource. In that case you can use some of the parameters passed to the fetch function to get creative.

##### Arguments
* [`id`] \(String, Object, or Undefined): The id that was passed into the action. May be a string if only one id needs to be specified, an object if multiple ids need to be specified (nested urls), or undefined if no ids need to be specified (simple collection). Useful if your unique identifiers are not confined to the url.
* [`actionType`] \(String): The type of CRUD action. Will be one of `LOAD`, `LOAD_COLLECTION`, `CREATE`, `UPDATE`, `DESTROY`.
* [`state`] \(Object): The redux state. Useful if you need to use some config held in state. Keep in mind that you should not use this to grab any unique identifiers as the id argument is what is used to cache.

### Reducer
```javascript
import { resourcesReducer } from 'iguazu-rest'
import { combineReducers, createStore } from redux;

const reducer = combineReducers({
  resources: resourcesReducer,
  // other reducers
})

const store = createStore(reducer);
```
### Actions
All iguazu-rest actions accept the same type of object as their single argument. The object can have the following properties:

* [`resource`] \(String): The name of the resource type, must match the key in resources config.
* [`id`] \(String, Object, or Undefined): The id or ids to be populated in the REST call. May be a string if only one id needs to be specified, an object if multiple ids need to be specified (nested urls), or undefined if no ids need to be specified (simple collection).
* [`opts`] \(Object or Undefined): fetch opts to be sent along with the REST call
* [`forceFetch`] \(Boolean or Undefined): forces the REST call to be made regardless of whether the resource or collection is loaded or not. Only applies to queryResource, queryCollection, loadCollection, and queryCollection.


#### Iguazu Actions
##### `queryResource({ resource, id, opts, forceFetch })`
##### `queryCollection({ resource, id, opts, forceFetch })`

These actions return an object following the iguazu pattern `{ data, status, promise}`.

**Example:**

```javascript
// Author.jsx
import { connectAsync } from 'iguazu';
import { queryResource } from 'iguazu-rest';
import BookList from './BookList';

const Author = ({ id, author: { name, bio } }) => (
  <div>
    <div>{name}</div>
    <div>{bio}</div>
    <BookList authorId={id} />
  </div>
);

function loadDataAsProps({ store: { dispatch }, ownProps: { id } }) {
  return {
    author: () => dispatch(queryResource({ resource: 'author', id }))
  };
}

export default connectAsync({ loadDataAsProps })(Author);

// BookList.jsx
import { connectAsync } from 'iguazu';
import { queryCollection } from 'iguazu-rest';

const BookList = ({ books }) => (
  <div>
    {books.map(book => <Book book={book} />)}
  </div>
);

function loadDataAsProps({ store: { dispatch }, ownProps: { authorId } }) {
  return {
    books: () =>
      dispatch(queryCollection( resource: 'book', id: { authorId } ))
  }
}

export default connectAsync({ loadDataAsProps })(BookList);
```

#### CRUD Actions
##### `loadResource({ resource, id, opts, forceFetch })`
##### `loadCollection({ resource, id, opts, forceFetch })`
##### `createResource({ resource, id, opts })`
##### `updateResource({ resource, id, opts })`
##### `destroyResource({ resource, id, opts })`

These actions return a promise that resolves with the fetched data on successful requests and reject with an error on unsuccessful requests. The error also contains the status and the body if you need to inspect those.

### Selectors
##### `getResource({ resource, id })(state)`
##### `getCollection({ resource, id, opts })(state)`

### Query opts
iguazu-rest allows you to specify your query parameters as part of the opts passed to fetch. If they are used to filter a collection, make sure they are passed in this way instead of adding them directly to the url because it is necessary for proper caching. All of the resources get normalized, so iguazu-rest needs to a way to cache which resources came back for a set of query parameters.

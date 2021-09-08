# Integrate Strapi and React-admin
- https://strapi.io/
- https://marmelab.com/react-admin/

# Create the strapi server
```
npx create-strapi-app server
cd server
npm run develop
```

# Create the react client
```
yarn create react-app client
cd client
yarn add react-admin ra-strapi-rest
yarn start
```

# Create the helper for Cookies in the Client
```
// helpers/Cookies

const Cookies = {
	getCookie: (name) => {
		const v = document.cookie.match('(^|;) ?' + name + '=([^;]*)(;|$)');
    		return v ? v[2] : null;
	},

	setCookie: (name, value, days) => {
		var d = new Date();
    		d.setTime(d.getTime() + 24*60*60*1000*days);
    		document.cookie = name + "=" + value + ";path=/;expires=" + d.toGMTString();
	},

	deleteCookie: (name) => {
		Cookies.setCookie(name, '', -1)
	}
};

export default Cookies;
```

# Create the helper for Cookies
```
// authProvider.js
import Cookies from './helpers/Cookies'

export default {

    login: ({ username, password }) => {
        const identifier = username // strapi expects 'identifier' and not 'username'
        const request = new Request('http://localhost:1337/auth/local', {
            method: 'POST',
            body: JSON.stringify({ identifier, password }),
            headers: new Headers({ 'Content-Type': 'application/json'})
        });
        return fetch(request)
            .then(response => {
                if (response.status < 200 || response.status >= 300) {
                    throw new Error(response.statusText);
                }
                return response.json();
            })
            .then(response => {
                Cookies.setCookie('token', response.jwt, 1);
                Cookies.setCookie('role', response.user.role.name, 1);
            });
    },

    logout: () => {
        Cookies.deleteCookie('token');
        Cookies.deleteCookie('role');
        return Promise.resolve();
    },

    checkAuth: () => {
        return Cookies.getCookie('token') ? Promise.resolve() : Promise.reject();
    },

    checkError: ({ status }) => {
        if (status === 401 || status === 403) {
            Cookies.deleteCookie('token');
            Cookies.deleteCookie('role');
            return Promise.reject();
        }
        return Promise.resolve();
    },

    getPermissions: () => {
        const role = Cookies.getCookie('role');
        return role ? Promise.resolve(role) : Promise.reject();
    },
}
```

# Create the post.js
```
// post.js
import React from 'react';
import { ReferenceInput, BooleanInput, SelectInput, List, Datagrid, TextField, TextInput, Edit, Create, SimpleForm } from 'react-admin';

const postFilters = [
    <TextInput source="q" label="Search" alwaysOn />,
    <ReferenceInput source="title" label="Title" reference="posts" allowEmpty>
        <SelectInput optionText="title" />
    </ReferenceInput>,
];

export const PostList = (props) => (
    <List filters={postFilters} {...props}>
        <Datagrid rowClick="edit">
            <TextField source="id" />
            <TextField source="title" />
            <TextField source="description" />
        </Datagrid>
    </List>
);

export const PostEdit = props => (
    <Edit {...props}>
        <SimpleForm>
            <TextInput disabled source="id" />
            <TextInput source="title" />
            <TextInput multiline source="description" />
            <BooleanInput source="published" />
        </SimpleForm>
    </Edit>
);


export const PostCreate = props => (
        <Create {...props}>
            <SimpleForm>
                <TextInput disabled source="id" />
                <TextInput source="title" />
                <TextInput multiline source="description" />
                <BooleanInput source="published" />
            </SimpleForm>
        </Create>
    );
```

# Create the App.js

```
import * as React from "react";
import { fetchUtils, Admin, Resource } from 'react-admin';
import jsonServerProvider from 'ra-data-json-server';
import simpleRestProvider from 'ra-strapi-rest';
import authProvider from './authProvider'
import Cookies from './helpers/Cookies';
import PostIcon from '@material-ui/icons/Book';

import { PostList, PostEdit, PostCreate } from './posts';

const httpClient = (url, options = {}) => {
  if (!options.headers) {
      options.headers = new Headers({ Accept: 'application/json' });
  }
  const token = Cookies.getCookie('token')
  options.headers.set('Authorization', `Bearer ${token}`);
  return fetchUtils.fetchJson(url, options);
}

const dataProvider = simpleRestProvider('http://localhost:1337', httpClient);

function App() {
  return (
    <Admin authProvider={authProvider} dataProvider={dataProvider}>
        <Resource name="posts" list={PostList} edit={PostEdit} create={PostCreate} icon={PostIcon}/>
    </Admin>
  );
}

export default App;
```

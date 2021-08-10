In this post, I'll demonstrate how to build a login screen with Google Sign-In, FaunaDB, and StepZen.

Here is the live-code demo video for this blog, https://www.youtube.com/watch?v=8nzJdgrZ7FQ.

## Getting Started

### Sign Up for StepZen

Create a [StepZen account](https://stepzen.com/request-invite) first, in order to get your API and admin keys (click the "My Account" button on the top right once you log in to see where they are).

Next, [install the StepZen CLI](https://stepzen.com/docs/quick-start) to be able to deploy your endpoint from the command line.

### Using The StepZen CLI

If this is your first time using the StepZen CLI, the first thing you'll need to do from the command line is run

```bash
npm install -g stepzen
stepzen login
```

You'll be prompted for your credentials. Use the ones from your account page:

```bash
What is your account name?: {ACCOUNT}
What is your admin key?: {ADMINKEY}
```

### Setting up the Expo React Native App

Expo is an easy to use react native platform that we will use for this project. 

First, install the CLI so we can jump start our app.

```bash
npm install --global expo-cli
```

Initialize the new app. When prompted, select `tabs (TypeScript), several example screens and tabs using react-navigation and TypeScript`.  This will install many dependencies that improve the react native navigation experience.

```bash
expo init faunadb-login-demo
cd faunadb-login-demo
```

### Setup Fauna Database

Go to [Fauna.com](https://fauna.com) and click Sign Up. You will be taken to the Fauna dashboard.

### Create Database

Create a new database in Fauna named `stepzen-fauna`. Use the "Classic" region group and be sure to check the "Use demo data" option.

Once the database is created, open the [GraphQL](https://dashboard.fauna.com/graphql/@db/global/stepzen-fauna) menu option within the Fauna Dashboard. You'll need the HTTP authorization header at the bottom of the Playground to use in your `config.yaml` file.

![04-http-headers](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/85rtr7kmp97zyiee9a0o.png)

Copy the schema below, save it as a `.graphql` file, and upload it to the Fauna GraphQL Playground by selecting Merge Schema. This will allow us to write the users that log in to our FaunaDB User Collection. 

```graphql
type User {
  email: String!
  familyName: String!
  givenName: String!
  id: String!
  name: String!
  photoUrl: String!
}

input UserInput {
  email: String!
  familyName: String!
  givenName: String!
  id: String!
  name: String!
  photoUrl: String!
}

type MutationUser {
  createUser(data: UserInput!): User!
}

type QueryUser {
  findUserByID(id: ID!): User
}
```

Now on your local machine, go to the root of your `faunadb-login-demo` project and create a StepZen folder. 

```bash
mkdir stepzen && cd stepzen
```

Create a `config.yaml` file in the folder with the following contents (replacing `Basic MY_FAUNA_KEY` with the information you just copied from the GraphQL Playground in the Fauna dashboard.

```yaml
configurationset:
  - configuration:
      name: fauna_config
      Authorization: Basic MY_FAUNA_KEY
```

Next, we need to create our GraphQL schema code that builds our API within StepZen.

### Deploy Your StepZen Endpoint

Next, create a file named `fauna.graphql` within the local working directory containing your `config.yaml`.

```graphql
type User {
  email: String!
  familyName: String!
  givenName: String!
  id: String!
  name: String!
  photoUrl: String!
}

input UserInput {
  email: String!
  familyName: String!
  givenName: String!
  id: String!
  name: String!
  photoUrl: String!
}

type Mutation {
  createUser(data: UserInput!): User!
    @graphql(
        endpoint: "https://graphql.us.fauna.com/graphql"
        configuration: "fauna_config"
        )
}

type Query {
    findUserByID(id: ID!): User
        @graphql (
            endpoint: "https://graphql.us.fauna.com/graphql"
            configuration: "fauna_config"
        )
}
```

Now, the endpoint `https://graphql.us.fauna.com/graphql` may change based on your location.  Be sure to add the correct endpoint in your faunaDB GraphQL Explorer. This example uses the @graphql stepzen directive, so StepZen will query the data the same way you would on the FaunaDB endpoint.

Add the `fauna.graphql` schema to an `index.graphql` file.

```graphql
schema
  @sdl(
    files: ["fauna.graphql"]
  ) {
  query: Query
}
```

Create an optional folder `stepzen.config.json`, to tell StepZen what to name the API endpoint. This is recommended for this tutorial

```json
{
  "endpoint": "demo/native-login"
}
```

Now your StepZen folder should have the file structure below.

```
.
├── stepzen
    ├── index.graphql
    ├── fauna.graphql
    ├── config.yaml
    └── stepzen.config.json
```

Using the StepZen CLI, run the following command.

```bash
stepzen start
```

The endpoint should spin up a `localhost:5000` environment with the following endpoint now deployed! 

```bash
http://localhost:5000/demo/native-login

Deploying to StepZen...... done

Successfully deployed demo/native-login

Your endpoint is available at https://youraccountname.stepzen.net/demo/native-login/__graphql
```

### Creating Our Google Sign-In ClientIds

First, create a google cloud account. There are free trials available.  

Once created, go to credentials in the dashboard and create two oAuth clientIds for android and ios applications. Copy and save these clientIds for when we install the dependency `expo-google-app-auth` in our react native app.

https://console.cloud.google.com/apis/credentials/oauthclient

#### Creating our React Native App

First, in the root of our project, let's install some dependencies. You can do npm or yarn.

```
yarn add @apollo/client expo-google-app-auth expo-blur react-native-reanimated
```

In our `App.tsx` file, we need to use the `@apollo/client` to add our StepZen endpoint and admin key to the frontend project.

```javascript
import {
	ApolloClient,
	ApolloProvider,
	createHttpLink,
	InMemoryCache,
} from "@apollo/client";

const client = new ApolloClient({
	link: createHttpLink({
		credentials: "same-origin",
		headers: {
			Authorization: `Apikey {{your_key}}`,
		},
		uri: "https://youraccountname.stepzen.net/demo/native-login/__graphql",
	}),
	cache: new InMemoryCache(),
});

return (
			<ApolloProvider client={client}>
				<SafeAreaProvider>
					<Navigation colorScheme={colorScheme} />
                    <StatusBar />
				</SafeAreaProvider>
			</ApolloProvider>
		);
```

It's that simple to connect a StepZen API to a react native frontend project. FaunaDB in our StepZen API is ready to be used in the react native app.

In the `App.tsx`, we are going to be navigating between the `LoginScreen` and `UserScreen`, so we need to add a `<NavigationContainer>` for both pages.

```javascript
export default function App() {
	const isLoadingComplete = useCachedResources();

	if (!isLoadingComplete) {
		return null;
	} else {
		return (
			<ApolloProvider client={client}>
				<SafeAreaProvider>
					<NavigationContainer>
						<Stack.Navigator initialRouteName="Login">
							<Stack.Screen 
								name=" " 
								component={LoginScreen}
								options={{
									headerTransparent: true,
									headerTitle: "Login/Logout",
									headerBackground: () => (
										<BlurView tint="light" intensity={100} />
										),
								}}
							/>
							<Stack.Screen 
								name="Home" 
								component={UserScreen} 
								options={{
									title: "Golf-Austin",
									headerTransparent: true,
									headerTitleAlign: "center",

									}}
							/>
						</Stack.Navigator>
					</NavigationContainer>
				</SafeAreaProvider>
			</ApolloProvider>
		);
	}
}
```

Moving on to the individual screens of our app, let's rename `TabOneScreen.tsx` to `LoginScreen.tsx` and add the following imports.

```javascript
import React, { useState } from "react";
import { StyleSheet, View, Button, ImageBackground, Image } from "react-native";
import * as Google from "expo-google-app-auth";
import { useMutation } from "@apollo/client";
import { CREATE_USER } from "../mutations/create-user"
```

Everything being imported is a dependency package except for `CREATE_USER`, which is the faunaDB mutation that will create our user. 

Let's go create that mutation file and add the following mutation

```javascript
import { gql } from "graphql-tag"

export const CREATE_USER = gql`
    mutation CreateUser ($name: String!, $photoUrl: String!, $email: String!, $familyName: String!, $givenName: String!, $id: String!) {
        createUser(
            data: {
                name: $name
                photoUrl: $photoUrl
                email: $email
                familyName: $familyName
                givenName: $givenName
                id: $id
            }
        ) {
            email
            givenName
            familyName
            name
            photoUrl
        }
    }
`;
```

The mutation is ready to be used in the log in process. Referring back to the clienIds created earlier in google cloud, we can write out our `LoginScreen` as below. Add your keys in the `Google.logInAsync`. 

```javascript
const LoginScreen = ({ navigation } : any) => {
    const [createUser, {data}] = useMutation(CREATE_USER)

    const [user, setUser] = useState(null);
    const [accessToken, setAccessToken] = useState();
    const signInAsync = async () => {
        try {
        const { type, accessToken, user } : any = await Google.logInAsync({
            iosClientId: `{{ add key }}`,
            androidClientId: `{{ add key }}`,
        });

        if (type === "success") {
            setUser(user);
            setAccessToken(accessToken);
            createUser({variables: {name: user.name, email: user.email,  familyName: user.familyName, givenName: user.givenName,  id: user.id, photoUrl: user.photoUrl}});
            navigation.navigate("Home", { user });
        }
        } catch (error) {
            console.log("LoginScreen.js 19 | error with login", error);
        }
    };

    const goBack = () => {
        navigation.navigate("Home", { user });
    }

    const image = { uri: 'https://images.unsplash.com/photo-1587174486073-ae5e5cff23aa?ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&ixlib=rb-1.2.1&auto=format&fit=crop&w=750&q=80' };

    return (
        <>
            <ImageBackground source={image} style={styles.image} blurRadius={0.3}>
                <View style={styles.buttonContainer}>
                    <Image
                        style={styles.tinyLogo}
                        source={{
                            uri: 'https://upload.wikimedia.org/wikipedia/commons/thumb/5/53/Google_%22G%22_Logo.svg/120px-Google_%22G%22_Logo.svg.png',
                        }}
                    />
                    {!user && <Button title="Login with Google" onPress={signInAsync} />}
                    {user && <Button title="Go Back" onPress={goBack} />
                    }
                </View>
            </ImageBackground>
        </>
    );
};

export default LoginScreen;
```

Then add some styling at the end of the file.

```javascript
const styles = StyleSheet.create({
    buttonContainer: {
        flex: 1,
        alignItems: 'center',
        padding: 60,
        backgroundColor: 'rgba(100,200,100,0.2)',
    },
    image: {
        flex: 1,
        resizeMode: 'cover',
        justifyContent: 'center',
        
    },
    tinyLogo: {
        width: 50,
        height: 50,
        marginBottom: 20,
        marginTop: 50,
    },
    profilePic: {
        flex: 0,
        width: 90,
        height: 90,
        // marginTop: 0,
        borderRadius: 250,
    }
});
```

Let's break it down what's happening in this function and then run the expo app.

OnPress of the Login Button, `<Button title="Login with Google" onPress={signInAsync} />`, The `signInAsync` takes the clientId from your google cloud account and returns a `user` that we assign to the state, `setUser(user)` and the same goes for assigning the `accessToken`. 

The StepZen mutation with FaunaDB occurs when we assign all the variables of `user` to `createUser`, the mutation that is written in our import folder, `import { CREATE_USER } from "../mutations/create-user"`.

```javascript
    const [createUser, {data}] = useMutation(CREATE_USER)

    const [user, setUser] = useState(null);
    const [accessToken, setAccessToken] = useState();

    const signInAsync = async () => {
        try {
        const { type, accessToken, user } : any = await Google.logInAsync({
            iosClientId: `{{ add key }}`,
            androidClientId: `{{ add key }}`,
        });

        if (type === "success") {
            setUser(user);
            setAccessToken(accessToken);
            createUser({variables: {name: user.name, email: user.email,  familyName: user.familyName, givenName: user.givenName,  id: user.id, photoUrl: user.photoUrl}});
            navigation.navigate("Home", { user });
        }
        } catch (error) {
            console.log("LoginScreen.js 19 | error with login", error);
        }
    };
```

Here is an example of the fields returned from Google Sign-In that we assign as variables to the StepZen mutation, `createUser`.

```json
{
  "email": "sam@stepzen.com",
  "familyName": "Hill",
  "givenName": "Samuel",
  "id": "1234",
  "name": "Samuel Hill",
  "photoUrl": "https://lh3.googleusercontent.com/a-/AOh14Gi3fCa_eM-k0vLZK5z1gChGA0RS_3C-OhyIp8ml=s96-c",
}
```

Now, one thing we still need to do is create the UserScreen.tsx so the user can see they successfully logged in.

Rename the `TabTwoScreen.tsx` to `UserScreen.tsx` and create the following component.

```javascript
import React from "react";
import { StyleSheet, Text, View, ImageBackground, Image } from "react-native";

const UserScreen = ({ route } : any) => {
const { user } = route.params;

const backgroundImage = { uri: 'https://images.unsplash.com/photo-1595841055318-943e15fbbe80?ixid=MnwxMjA3fDB8MHxzZWFyY2h8MTgzfHxnb2xmfGVufDB8fDB8fA%3D%3D&ixlib=rb-1.2.1&auto=format&fit=crop&w=400&q=60' };

    return (
        <>
            <ImageBackground source={backgroundImage} style={styles.image}>
                    <View style={styles.container}>
                        <View>
                            <Image
                                style={styles.profilePic}
                                source={{
                                    uri: `${user.photoUrl}`,
                                }}
                            />
                        </View>
                        <Text style={styles.banner}>Welcome {user.name}</Text>
                    </View>
            </ImageBackground>
        </>
    );
};

export default UserScreen;
```

And add some styling at the end of the file.

```javascript
const styles = StyleSheet.create({
    container: {
        flex: 1,
        flexDirection: 'column',
        justifyContent: 'space-between',
        alignItems: 'center',
        textAlign: 'center',
        height: '100%',
    },
    image: {
        flex: 1,
        resizeMode: 'contain',
        justifyContent: 'center',
    },
    profilePic: {
        flex: 0,
        width: 90,
        height: 90,
        marginTop: 80,
        borderRadius: 250,
    },
    banner: {
        flex: 1,
        fontSize: 38
    }
});
```

Heading back over to our `LoginScreen`, we can now see two functions that we put in earlier should work to perfection when we have a successful login.

This `navigate` function will redirect the user on successful login.

```javascript
const signInAsync = async () => {
        try {
            if (type === "success") {
                navigation.navigate("Home", { user });
            }
    };
```

The `goBack` function runs when pressing the `<Button title="Go Back" onPress={goBack}` button on the Login page.

```javascript
const goBack = () => {
        navigation.navigate("Home", { user });
    }
```

### Run the app

Now with a fully created `App.tsx` with our StepZen endpoint and apikey, a `LoginScreen` with our google clientIds, and a `UserScreen` for successful logins, we can run the app!

In your terminal, run the following command.

```bash
expo start
```

Download the expo app on your phone, and scan the QR code provided at `http://localhost:19002/` on your computer.

Select the "Login with Google" button on the app, and when prompted, login with your google credentials. On login, you should be redirected to the `UserScreen`, and the faunaDB User Collection should be updated with the login credentials! 

Here is an example of a user added to FaunaDB.

```
{
  "ref": Ref(Collection("User"), "306547471891300420"),
  "ts": 1628605300720000,
  "data": {
    "familyName": "Hill",
    "name": "Sam Hill",
    "givenName": "Sam",
    "photoUrl": "https://lh3.googleusercontent.com/a-/AOh14Gi3fCa_eM-k0vLZK5z1gChGA0RS_3C-OhyIp8ml=s96-c",
    "email": "sam@stepzen.com",
    "id": "1234"
  }
}
```

Now you successfully have a google sign-in with a database that can store all the users. This demonstrates the ease of adding a FaunaDB to your single StepZen endpoint that can be combined with any other data source. Writing the data of the user to more than one database or API can easily be added to this configuration in the StepZen schema.

## Where To Go From Here

To learn more on how to use FaunaDB and mutations, you can use [our docs](https://stepzen.com/docs/using-graphql/graphql-mutation-basics).

The StepZen docs also hold information on connecting other backends to your endpoint, like [REST APIs](https://stepzen.com/docs/connecting-backends/how-to-connect-a-rest-service) and [databases like PostGresQL](https://stepzen.com/docs/connecting-backends/how-to-connect-a-postgresql-database).

If you've got more questions, [hit us up on Discord](https://discord.com/channels/768229795544170506/768229795544170509), we'd love to chat. ;)

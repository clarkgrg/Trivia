# Creating a Trivia App with Ignite CLI - Part I (updated)

This is an updated version of Robin Heinze's excellent [blog](https://shift.infinite.red/creating-a-trivia-app-with-ignite-bowser-part-1-1987cc6e93a1) post on the [ignite CLI](https://github.com/infinitered/ignite). This post is copied directly from her work with only slight changes reflecting the updated Ignite version.

In this post, we’ll go through the steps of creating the data model for a trivia app to illustrate how to use our stack with real examples.

(Psst, spoiler alert! If you’d like to follow along with real live code, check out the finished app!)

First things first, Install ignite-cli if you haven’t already!

```
npm install -g ignite-cli
```

Now let’s Ignite our new app!

```
ignite new Trivia
```

After Ignite finishes running, run `npx react-native run-ios` or `npx react-native run android` and you should see something like this:

![Ignite Main](https://miro.medium.com/max/1400/1*loYwHRaQotXoRb0AHVI7Qg.png)

## Defining Our Models

Next, let’s start building our data models. We’re going to use The [Open Trivia Database](https://opentdb.com/) as our data source, so we’ll build our models to match that data.

### Question

We need a question model! Each question has the following properties:

`id, category, type, difficulty, question, correctAnswer, incorrectAnswers`

Let’s create the MobX State Tree (MST) model:

```
ignite generate model question
```

This will create a folder `app/models/question/` which contains a few files:

```
question.ts
question.test.ts
```

We won’t be writing unit tests quite yet, so we’ll focus on `question.ts`

![question.ts](https://github.com/clarkgrg/Trivia/blob/main/assets/question.ts.png?raw=true)

This file has a couple different parts. First there’s `QuestionModel`, which is the heart of this file. It’s the MST model definition. Each model instance has properties, like a regular JS object, but it also has views (computed values), and actions.

At the bottom of the file, we define some TypeScript interfaces which will help TypeScript to know about the shape of our data as we use it throughout the app.

We’ll focus on properties first, so let’s add the properties we want each `Question` to have:

```Javascript
export const QuestionModel = types
  .model("Question")
  .props({
    id: types.identifier,
    category: types.maybe(types.string),
    type: types.enumeration(["multiple", "boolean"]),
    difficulty: types.enumeration(["easy", "medium", "hard"]),
    question: types.maybe(types.string),
    correctAnswer: types.maybe(types.string),
    incorrectAnswers: types.optional(types.array(types.string), []),
  })
  .views((self) => ({})) // eslint-disable-line @typescript-eslint/no-unused-vars
  .actions((self) => ({})) // eslint-disable-line @typescript-eslint/no-unused-vars
```

Let’s unpack this a bit.

We have `id`, which is `types.identifier`. This is MST’s version of a primary key. It has to be unique, and it allows you to do some helpful things like reference your model instance by only its identifier. You can read a bit more about identifiers [here](https://github.com/mobxjs/mobx-state-tree#identifiers).

Then, we have some string properties: `category`, `question`, and `correctAnswer`. These are all free-form text values, and we’ve made them optional by using `types.maybe()`. That means these values can be undefined.

We also have a property `incorrectAnswers`, which is an array of strings. Instead of `types.maybe()`, we’ve used `types.optional()`. This allows us to not specify a value for this property, but instead of being undefined, it will take a default value of `[]`.

Finally, we have `type` and `difficulty` which is are both enumerations. This forces the value to be one of a predetermined list of values, in this case strings.

### Question Store

Since we’ll be listing more than one trivia question, we need a place to store a list of them. So let’s create QuestionStore.

```
ignite generate model question-store
```

IGNITE TIP 🎉: If you Ignite apps as often as we do, you can abbreviate `generate` as `g`, like `ignite g model <model-name>`

This file looks a lot like `question.ts` did when we first generated it, but we’re going to add different properties. For now, it only has one: an array of type `QuestionModel`. This will be an array of MST instances. We may decide we need more properties later, but this will be enough to get started.

```Javascript
import { QuestionModel } from "../question/question"

/**
 * Model description here for TypeScript hints.
 */
export const QuestionStoreModel = types
  .model("QuestionStore")
  .props({
    questions: types.optional(types.array(QuestionModel), []),
  })
  .views((self) => ({})) // eslint-disable-line @typescript-eslint/no-unused-vars
  .actions((self) => ({})) // eslint-disable-line @typescript-eslint/no-unused-vars
```

## Hooking up the API

Okay, we have some models describing the shape of data that we want. Now let’s get some data.

### API Service

Ignite comes with an API service all ready to customize, so let’s customize it.

Changes to `api-config.ts`

```Javascript
const API_URL = "https://opentdb.com/api.php"
...
export const DEFAULT_API_CONFIG: ApiConfig = {
  url: API_URL,
  timeout: 10000,
}
```

Changes to `api.types.ts`

```Javascript
import { QuestionSnapshot } from "../../models/question"

export type GetQuestionsResult = { kind: "ok"; questions: QuestionSnapshot[] } | GeneralApiProblem
```

Changes to `api.ts`

```Javascript
const API_PAGE_SIZE = 10

const convertQuestion = (raw: any): QuestionSnapshot => {
  const id = uuid.v1()
  return {
    id: id,
    category: raw.category,
    type: raw.type,
    difficulty: raw.difficulty,
    question: raw.question,
    correctAnswer: raw.correct_answer,
    incorrectAnswers: raw.incorrect_answers,
  }
}

export class Api {
...

  /**
   * Gets a list of trivia questions.
   */
  async getQuestions(): Promise<Types.GetQuestionsResult> {
    // make the api call
    const response: ApiResponse<any> = await this.apisauce.get("", { amount: API_PAGE_SIZE })

    // the typical ways to die when calling an api
    if (!response.ok) {
      const problem = getGeneralApiProblem(response)
      if (problem) return problem
    }

    // transform the data into the format we are expecting
    try {
      const rawQuestions = response.data.results
      const convertedQuestions: QuestionSnapshot[] = rawQuestions.map(convertQuestion)
      return { kind: "ok", questions: convertedQuestions }
    } catch (e) {
      __DEV__ && console.tron.log(e.message)
      return { kind: "bad-data" }
    }
  }
}
```

We need to install `react-native-uuid`.

NOTE: when I installed this package it broke my build. I had to remove node_modules, package-lock.json and run `npm install`.

```Javascript
npm install react-native-uuid
```

Here’s what we did:

- We updated the base URL in `app/services/api/api-config.ts`

- We removed the getUsers and getUser examples in `app/services/api/api.ts` and added getQuestions

- Updated the function return types in `app/services/api/api.types.ts` to use `QuestionSnapshot`. Snapshots are the serialized version of MST models, without all the dynamic features like views, and actions. You can think of snapshots as the immutable, plain JS object version of your MST model.

### Fetching the Data

Now that our API service is setup, let’s tell our models how to call the API and populate data into our store.

It’s time to make our first MST `action`.

But first, let’s add one thing to our Question Store model.

```Javascript
import { withEnvironment } from "../extensions/with-environment"

export const QuestionStoreModel = types
  .model("QuestionStore")
  .props({
    questions: types.optional(types.array(QuestionModel), []),
  })
  .extend(withEnvironment)
  .views(self => ({})) // eslint-disable-line @typescript-eslint/no-unused-vars
  .actions(self => ({})) // eslint-disable-line @typescript-eslint/no-unused-vars
```

Notice we’ve added the line `.extend(withEnvironment)`. Extensions are a neat way of sharing commonly used patterns in MST. In this case, it adds a view to the model called environment which calls MST’s getEnv function, and returns the contents of our environment (located in `app/models/environment.ts`). Check out the [docs](https://mobx-state-tree.js.org/concepts/dependency-injection) to learn more about getEnv and dependency injection in MST.

Now, let’s get back to our action.

```Javascript
/**
 * Model description here for TypeScript hints.
 */
export const QuestionStoreModel = types
  .model("QuestionStore")
  .props({
    questions: types.optional(types.array(QuestionModel), []),
  })
  .extend(withEnvironment)
  .views((self) => ({})) // eslint-disable-line @typescript-eslint/no-unused-vars
  .actions((self) => ({
    saveQuestions: (questionSnapshots: QuestionSnapshot[]) => {
      const questionModels: Question[] = questionSnapshots.map((c) => QuestionModel.create(c)) // create model instances from the plain objects
      self.questions.replace(questionModels) // Replace the existing data with the new data
    },
  }))
  .actions((self) => ({
    getQuestions: flow(function* () {
      const result: GetQuestionsResult = yield self.environment.api.getQuestions()

      if (result.kind === "ok") {
        self.saveQuestions(result.questions)
      } else {
        __DEV__ && console.tron.log(result.kind)
      }
    }),
  })) // eslint-disable-line @typescript-eslint/no-unused-vars
```

We’ve added two actions:

- `saveQuestions` takes an array of snapshots and turns them into model instances and then saves them to the `questions` array.

- `getQuestions` calls the API method to get the array of snapshots from the API, and then passes them to `saveQuestions`

The reason for using two `.action` blocks has to do with the timing of when actions are added to self. In order to call the action saveQuestions on self, we needed to have already defined it in a different .action block. See the docs for bit more information about this subtle gotcha.

That’s it for Part I! We now have a solid data model, and a way to populate our store from an external API. Next time, in Part II, we’ll consume our data by hooking up a UI!

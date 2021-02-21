# Creating a Trivia App with Ignite Bowser — Part II (updated)

This is an updated version of Robin Heinze's excellent blog post on the [ignite CLI](https://shift.infinite.red/creating-a-trivia-app-with-ignite-bowser-part-ii-a286a978d0c8). This post is copied directly from her work with only slight changes reflecting the updated changes.

In Part I, we setup a data model for our Trivia App based on [Open Trivia DB](https://opentdb.com/) and built using Ignite Bowser. In Part II, we’re going to hook up some screens to complete the data lifecycle!

First things first, we need a screen. Let’s generate one:

```
ignite g screen question
```

Now we have a blank screen called `question-screen.tsx`. Add this screen to `app/screens/index.ts` and remove `demo` and `welcome`.

```Javascript
export * from "./question/question-screen"
```

Let’s replace the welcome and demo screens that come with Bowser with this new screen. Now our `app/navigation/main-navigator.tsx` should look like this:

```Javascript
import { createStackNavigator } from "@react-navigation/stack"
import { QuestionScreen } from "../screens"

/**
 * This type allows TypeScript to know what routes are defined in this navigator
 * as well as what properties (if any) they might take when navigating to them.
 *
 * If no params are allowed, pass through `undefined`. Generally speaking, we
 * recommend using your MobX-State-Tree store(s) to keep application state
 * rather than passing state through navigation params.
 *
 * For more information, see this documentation:
 *   https://reactnavigation.org/docs/params/
 *   https://reactnavigation.org/docs/typescript#type-checking-the-navigator
 */
export type PrimaryParamList = {
  question: undefined
}

// Documentation: https://reactnavigation.org/docs/stack-navigator/
const Stack = createStackNavigator<PrimaryParamList>()

export function MainNavigator() {
  return (
    <Stack.Navigator
      screenOptions={{
        headerShown: false,
      }}
    >
      <Stack.Screen name="question" component={QuestionScreen} />
    </Stack.Navigator>
  )
}
```

Before we run the app, let’s fix up a few things:

- Add an entry in `app/i18n/en.json` for this screen’s text translations:

```Javascript
"questionScreen": {
  "header": "Questions"
}
```

- Add an enum for spacing sizes in `app/theme/spacing`.

```Javascript
export enum sizes {
  none = 0,
  tiny = 1,
  smaller = 2,
  small = 3,
  medium = 4,
  mediumPlus = 5,
  large = 6,
  huge = 7,
  massive = 8,
}
```

- Add `darkPlum: "#20162D",` to `app/theme/palette.ts` and change the background to use darkPlum in `app/theme/color.ts`.

- Change our question-screen as follows:

```Javascript
import React from "react"
import { observer } from "mobx-react-lite"
import { View, ViewStyle } from "react-native"
import { Screen, Text } from "../../components"
// import { useNavigation } from "@react-navigation/native"
// import { useStores } from "../../models"
import { color, spacing, sizes } from "../../theme"

const ROOT: ViewStyle = {
  flex: 1,
  backgroundColor: color.background,
  paddingHorizontal: spacing[sizes.large],
}

const HEADER_CONTAINER: ViewStyle = {
  marginTop: spacing[sizes.huge],
  marginBottom: spacing[sizes.medium],
}

export const QuestionScreen = observer(function QuestionScreen() {
  // Pull in one of our MST stores
  // const { someStore, anotherStore } = useStores()

  // Pull in navigation via hook
  // const navigation = useNavigation()
  return (
    <Screen style={ROOT} preset="scroll">
      <View style={HEADER_CONTAINER}>
        <Text preset="header" tx="questionScreen.header" />
      </View>
    </Screen>
  )
})
```

When we run the app, it looks like this:

<QUESTIONS SCREEN>

## Adding Data

Our screen looks a little barren, so let’s add some data!

First, we’ll add our `QuestionStore` to the `app/models/root-store/root-store.ts`.

```Javascript
import { Instance, SnapshotOut, types } from "mobx-state-tree"
import { QuestionStore, QuestionStoreModel } from "../question-store/question-store"

/**
 * A RootStore model.
 */
// prettier-ignore
export const RootStoreModel = types.model("RootStore").props({
  questionStore: types.optional(QuestionStoreModel, {} as QuestionStore)
})

/**
 * The RootStore instance.
 */
export interface RootStore extends Instance<typeof RootStoreModel> {}

/**
 * The data of a RootStore.
 */
export interface RootStoreSnapshot extends SnapshotOut<typeof RootStoreModel> {}
```

Now we can use our questionStore in our screen.

```Javascript
import { useStores } from "../../models"
...
export const QuestionScreen = observer(function QuestionScreen() {
  // Pull in one of our MST stores
  const { questionStore } = useStores()
...
```

### Making a List

Now let’s pull our data out of the store and make a list!

- First track whether we are refreshing the data with a useState() hook from react.

- Next lets install `lodash.shuffle`.

```Javascript
npm i --save lodash.shuffle
```

- Add a view called `allAnswers` to `question.ts` that makes an array containing all of the answers (incorrect and correct) and then shuffles them using `lodash.shuffle`. We use this in renderItem to display the answer options in random order.

```Javascript
  .views((self) => ({
    get allAnswers() {
      return shuffle(self.incorrectAnswers.concat([self.correctAnswer]))
    },
```

- Add a function `decodeHTMLEntities()` in new file `app/utils/html-decode.ts`. This will decode both the questions and the answers making them show correctly.

- When we pull-to-refresh, we load a new set of questions. A `useEffect()` hook is added to fetch questions initially.

- We added an extraData prop to our FlatList. This a common gotcha in MST. In order for the FlatList (a pure component) to know to update when we get new data in our list, we have to explicitly specify a value that we know will change (in our case, the text of the first question).

- Change the preset on the `<Screen />` component to `fixed`. This removes the warnings on having a `FlatList` inside a `ScrollView`.

When we run the app, the screen looks like this:

<SCREEN PIC HERE>

Here is our udpated `question-screen.tsx`

```Javascript
import React, { useEffect, useState } from "react"
import { observer } from "mobx-react-lite"
import { FlatList, TextStyle, View, ViewStyle } from "react-native"
import { Screen, Text } from "../../components"
import { color, spacing, sizes } from "../../theme"
import { useStores } from "../../models"
import { Question } from "../../models/question/question"
import { decodeHTMLEntities } from "../../utils/html-decode"

const ROOT: ViewStyle = {
  flex: 1,
  backgroundColor: color.background,
  paddingHorizontal: spacing[sizes.large],
}

const HEADER_CONTAINER: ViewStyle = {
  marginTop: spacing[sizes.huge],
  marginBottom: spacing[sizes.medium],
}

const QUESTION_LIST: ViewStyle = {
  marginBottom: spacing[sizes.large],
}

const QUESTION_WRAPPER: ViewStyle = {
  borderBottomColor: color.line,
  borderBottomWidth: 1,
  paddingVertical: spacing[sizes.large],
}

const QUESTION: TextStyle = {
  fontWeight: "bold",
  fontSize: 16,
  marginVertical: spacing[sizes.medium],
}

const ANSWER: TextStyle = {
  fontSize: 12,
}

const ANSWER_WRAPPER: ViewStyle = {
  paddingVertical: spacing[sizes.smaller],
}

export const QuestionScreen = observer(function QuestionScreen() {
  // Are we refreshing the data
  const [refreshing, setRefreshing] = useState(false)

  // Pull in one of our MST stores
  const { questionStore } = useStores()
  const { questions } = questionStore

  useEffect(() => {
    fetchQuestions()
  }, [])

  const fetchQuestions = () => {
    setRefreshing(true)
    questionStore.getQuestions()
    setRefreshing(false)
  }

  const renderQuestion = ({ item }) => {
    const question: Question = item
    return (
      <View style={QUESTION_WRAPPER}>
        <Text style={QUESTION} text={decodeHTMLEntities(question.question)} />
        <View>
          {question.allAnswers.map((a, index) => {
            return (
              <View key={index} style={ANSWER_WRAPPER}>
                <Text style={ANSWER} text={decodeHTMLEntities(a)} />
              </View>
            )
          })}
        </View>
      </View>
    )
  }
  return (
    <Screen style={ROOT} preset="fixed">
      <View style={HEADER_CONTAINER}>
        <Text preset="header" tx="questionScreen.header" />
      </View>
      <FlatList
        style={QUESTION_LIST}
        data={questionStore.questions}
        renderItem={renderQuestion}
        extraData={{ extraDataForMobX: questions.length > 0 ? questions[0].question : "" }}
        keyExtractor={(item) => item.id}
        onRefresh={fetchQuestions}
        refreshing={refreshing}
      />
    </Screen>
  )
})
```

## Making It Interactive

Great! We have a list of questions now. But trivia isn’t very fun if you can’t answer the questions and see if you were right…

So let’s make our screen interactive.

First, we’re going to add a few things to our Question model:

```Javascript
import { Instance, SnapshotOut, types } from "mobx-state-tree"
import shuffle from "lodash.shuffle"

/**
 * Model description here for TypeScript hints.
 */
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
    guess: types.optional(types.string, ""),
  })
  .views((self) => ({
    get allAnswers() {
      return shuffle(self.incorrectAnswers.concat([self.correctAnswer]))
    },
    get isCorrect() {
      return self.guess === self.correctAnswer
    },
  })) // eslint-disable-line @typescript-eslint/no-unused-vars
  .actions((self) => ({
    setGuess(guess: string) {
      self.guess = guess
    },
  })) // eslint-disable-line @typescript-eslint/no-unused-vars
```

- A new property called guess, which is an optional string.

- We can’t change the value of properties using self. except from within the model itself, so in order to change guess from our screen, we need a setter action that we’ll call setGuess.

- A computed value (view) called `isCorrect` that checks the guess against correctAnswer. As long as neither of those values change, the return value of `isCorrect` will be cached and doesn’t need to be calculated every time it’s called.

Now we’re finally ready to make our questions interactive:

- Install react-native-radio-buttons. This package is deprecated but works for our example.

```Javascript
npm i -S react-native-radio-buttons
```

Make the following changes to `question-screen.tsx`:

```Javascript
const CHECK_ANSWER: ViewStyle = {
  paddingVertical: spacing[sizes.smaller],
  backgroundColor: color.palette.angry,
  marginTop: spacing[sizes.smaller],
}

export const QuestionScreen = observer(function QuestionScreen() {
  ...
  const onPressAnswer = (question: Question, guess: string) => {
    question.setGuess(guess)
  }

  const checkAnswer = (question: Question) => {
    if (question.isCorrect) {
      Alert.alert("That is correct!")
    } else {
      Alert.alert(`Wrong! The correct answer is: ${question.correctAnswer}`)
    }
  }

  const renderAnswer = (answer: string, selected: boolean, onSelect: () => void, index) => {
    const style: TextStyle = selected ? { fontWeight: "bold", fontSize: 14 } : {}
    return (
      <TouchableOpacity key={index} onPress={onSelect} style={ANSWER_WRAPPER}>
        <Text style={{ ...ANSWER, ...style }} text={decodeHTMLEntities(answer)} />
      </TouchableOpacity>
    )
  }

  const renderQuestion = ({ item }) => {
    const question: Question = item
    return (
      <View style={QUESTION_WRAPPER}>
        <Text style={QUESTION} text={decodeHTMLEntities(question.question)} />
        <RadioButtons
          options={question.allAnswers}
          onSelection={(guess) => onPressAnswer(question, guess)}
          selectedOption={question.guess}
          renderOption={renderAnswer}
        />
        <Button style={CHECK_ANSWER} onPress={() => checkAnswer(question)} text={"Check Answer!"} />
      </View>
    )
  }
  ...
}
```

Here’s what we did:

- We brought in [react-native-radio-buttons](https://github.com/ArnaudRinquin/react-native-radio-buttons) to add a radio button selector. The options that we pass to it are from the view we made earlier: `questions.allAnswers`

- When each radio button item is pressed, we set `guess` on that question to the text of the selected answer

- When the “Check Answer” button is pressed, we call our `isCorrect` computed value, and put up an alert telling the user whether they got it right or not.

Here’s our app in action!

<APP PICS HERE>

🎉 And there you have it! A basic Trivia app built with Ignite Bowser, using MobX State Tree and TypeScript.

You can view the complete source code here: https://github.com/robinheinze/ignite-trivia

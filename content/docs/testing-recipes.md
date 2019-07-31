---
id: testing-recipes
title: Test recipes 
permalink: docs/testing-recipes.html
---

This document lists some common testing patterns for React components.

- This assumes tests are running in jest. For details on setting up a testing environment, read [the Environments doc](/docs/testing-environments.html).
- This assumes you're using [React Testing Library](https://testing-library.com/react) (`@testing-library/react`) and [jest-dom](https://testing-library.com/jest-dom) (`@testing-library/jest-dom`) which is recommended
- This guide uses React's [`act()`](/docs/act.html) testing helper when using jest fake timers.

### Rendering {#rendering}

Testing whether a component renders correctly for given props is a useful signal to the stability of a component. Consider a simple component that renders a message based on a prop.

```jsx
// hello.js
export default function Hello(props) {
  return props.name === undefined ? (
    <span>Hey, stranger</span>
    ) : (
    <h1>Hello, {props.name}!</h1>
  );
}
```

We can write a test for this component:

```jsx
// hello.test.js
import "@testing-library/react/cleanup-after-each";
import "@testing-library/jest-dom/extend-expect";
import { render } from "@testing-library/react";
import Hello from "./hello";

test("renders without a name", () => {
  const { container } = render(<Hello />);
  expect(container).toHaveTextContent("Hey, stranger");
});

test("renders with a name", () => {
  const { container } = render(<Hello name="Sophie" />);
  expect(container).toHaveTextContent("Hello, Sophie!");
});
```

> Note: React Testing Library renders your components to a div which it appends to `document.body`. The import of `@testing-library/react/cleanup-after-each` automatically removes those after each test. The `@testing-library/jest-dom/extend-expect` adds helpful DOM-specific assertions like `toHaveTextContent`. It's best to configure Jest to include these files automatically via the `setupFilesAfterEnv` option (or you can import them in your `src/setupTests.js` file in create-react-app).

### Data fetching {#data-fetching}

Instead of calling real APIs in all your tests, you can mock requests with dummy data. Mocking data fetching with 'fake' data prevents flaky tests due to an unavailable backend, and makes them run faster. Note: you may still want to run a subset of tests using an "end-to-end" framework that tells whether the whole app is working together.

```jsx
// user.js
export default function User(props) {
  const [user, setUser] = useState(null);
  async function fetchUserData(id) {
    const response = await fetch("/" + id);
    setUser(await response.json());
  }
  useEffect(() => {
    fetchUserData(props.id);
  }, [props.id]);
  return user === null ? (
    <div>loading...</div>
  ) : (
    <details>
      <summary aria-label="name">{user.name}</summary>
      <strong aria-label="age">{user.age}</strong> years old
      <br />
      lives in <span aria-label="address">{user.address}</span>
    </details>
  );
}
```

We can write tests for it:

```jsx
// user.test.js
import { render } from "@testing-library/react";
import User from "./user";

beforeEach(() => {
  // mock window.fetch so the unit tests don't actually make HTTP requests
  jest.spyOn(window, "fetch").mockImplementation(() => {});
});

afterEach(() => {
  // remove the mock to ensure tests are completely isolated
  window.fetch.mockRestore();
});

test("renders user data", async () => {
  const fakeUser = {
    name: "Joni Baez",
    age: "32",
    address: "123, Charming Avenue",
  };
  window.fetch.mockImplementationOnce(() =>
    Promise.resolve({
      json: () => Promise.resolve(fakeUser),
    }),
  );

  const { getByLabelText, getByText } = render(<User id="123" />);

  expect(window.fetch).toHaveBeenCalledTimes(1);
  expect(window.fetch).toHaveBeenCalledWith("/123");

  await waitForElementToBeRemoved(() => getByText(/loading/i));

  expect(getByLabelText(/name/i)).toHaveTextContent(fakeUser.name);
  expect(getByLabelText(/age/i)).toHaveTextContent(fakeUser.age);
  expect(getByLabelText(/address/i)).toHaveTextContent(fakeUser.address);
});
```

### Mocking modules {#mocking-modules}

Some modules might not work well inside a testing environment, or may not be as essential to the tests itself. Mocking out these modules with different versions can make it easier to write tests for your own code. In this example, a `Contact` component embeds a GoogleMaps component from [`@react-google-maps/api`](https://react-google-maps-api-docs.netlify.com/)

```jsx
// map.js
import React from "react";
import { LoadScript, GoogleMap } from "react-google-maps";
export default function Map(props) {
  return (
    <LoadScript id="script-loader" googleMapsApiKey="YOUR_API_KEY">
      <GoogleMap id="example-map" center={props.center} />
    </LoadScript>
  );
}

// contact.js
import React from "react";
import Map from "./map";
export default function Contact(props) {
  return (
    <div>
      <address>
        Contact {props.name} via <a href={"mailto:" + props.email}>email</a>
        or on their <a href={props.site}>website</a>.
      </address>
      <Map center={props.center} />
    </div>
  );
}
```

In a testing environment, it's not very useful to load the Map component, besides also being inessential to testing our own code. In this situation, we can mock out the dependency itself to a dummy component, and run our tests.

```jsx
// contact.test.js
import React from "react";
import { render } from "@testing-library/react";
import Contact from "./contact";
import MockedMap from "./map";

jest.mock("./map", () => jest.fn(() => "DummyMap"));

test("renders contact information", () => {
  const center = { lat: 0, lang: 0 };
  render(<Contact name="" email="" site="" center={center} />);
  // ensure the mocked map component function was called correctly
  expect(MockedMap).toHaveBeenCalledWith({ center }, {});
});
```

### Events {#events}

Dispatch real events on components and elements, and observe the side effects of those actions on UI elements. Consider a `Toggle` component:

```jsx
// toggle.js
import React, { useState } from "react";
export default function Toggle({initial = false, onChange}) {
  const [state, setState] = useState(initial);
  return (
    <button
      onClick={() => {
        setState(previousState => !previousState);
        onChange(!state);
      }}
    >
      {state === true ? "Turn off" : "Turn on"}
    </button>
  );
}
```

We could write tests for it:

```jsx
// toggle.test.js
import React from "react";
import { render, fireEvent } from "@testing-library/react";
import Toggle from "./toggle";

test("changes value when clicked", () => {
  const handleChange = jest.fn();
  const { getByText } = render(
    <Toggle initial={true} onChange={handleChange} />,
  );
  const button = getByText(/turn off/i);
  fireEvent.click(button);
  expect(handleChange).toHaveBeenCalledTimes(1);
  expect(button).toHaveTextContent(/turn on/i);
  for (let i = 0; i < 5; i++) {
    fireEvent.click(button);
  }
  expect(handleChange).toHaveBeenCalledTimes(6);
  expect(button).toHaveTextContent(/turn off/i);
});
```

### Timers {#timers}

UIs could use timer based functions like `setTimeout` to run time sensitive code. In this example, a multiple choice panel waits for a selection and advances, timing out if a selection isn't made in 5 seconds.

```jsx
// card.js
import React, { useEffect } from "react";

export default function Card({ onSelect }) {
  useEffect(() => {
    const timeoutID = setTimeout(() => {
      onSelect(null);
    }, 5000);
    return () => {
      clearTimeout(timeoutID);
    };
  }, [onSelect]);
  return [1, 2, 3, 4].map(choice => (
    <button
      key={choice}
      data-testid={`card-button-${choice}`}
      onClick={() => onSelect(choice)}
    >
      {choice}
    </button>
  ));
}
```

We can write tests for this component by leveraging jest's timer mocks, and testing the different states it can be in.

> Note: When using Jest timers, you can't rely on React Testing Library calling [`act()`](/docs/act.html) for you, so you have to call it manually. As a convenience, `@testing-library/react` exports `act`.

```jsx
// card.test.js
import React from "react";
import { render, fireEvent, act } from "@testing-library/react";
import Card from "./card";

jest.useFakeTimers();

test("selects null after timing out", () => {
  const handleSelect = jest.fn();
  render(<Card onSelect={handleSelect} />);
  act(() => {
    jest.advanceTimersByTime(100);
  });
  expect(handleSelect).not.toHaveBeenCalled();
  act(() => {
    jest.advanceTimersByTime(10000);
  });
  expect(handleSelect).toHaveBeenCalledTimes(1);
  expect(handleSelect).toHaveBeenCalledWith(null);
});

test("cleans up on being removed", () => {
  const handleSelect = jest.fn();
  const { unmount } = render(<Card onSelect={handleSelect} />);
  unmount();
  act(() => {
    jest.advanceTimersByTime(10000);
  });
  expect(handleSelect).not.toHaveBeenCalled();
});

test("accepts selections", () => {
  const handleSelect = jest.fn();
  const { getByTestId } = render(<Card onSelect={handleSelect} />);

  fireEvent.click(getByTestId("card-button-2"));

  expect(handleSelect).toHaveBeenCalledTimes(1);
  expect(handleSelect).toHaveBeenCalledWith(2);
});
```

### Snapshot testing {#snapshot-testing}

Frameworks like Jest also let you save 'snapshots' of data, like DOM nodes. With snapshot testing, we can save DOM structures, and verify they don't change in the future. It's typically better to make more specific assertions than to use snapshots (or at least ensure that the snapshot is small) because these kinds of tests include implementation details meaning they break easily and teams can get desensitized to snapshot breakages.

```jsx
// hello.test.js, again
import React from "react";
import { render } from "@testing-library/react";
import Hello from "./hello";

test("preserves its structure", () => {
  expect(render(<Hello />).container).toMatchInlineSnapshot(`
    <div>
      <span>
        Hey, stranger
      </span>
    </div>
  `);

  expect(render(<Hello name="Sophie" />).container).toMatchInlineSnapshot(`
    <div>
      <h1>
        Hello, 
        Sophie
        !
      </h1>
    </div>
  `);
});
```

- [Jest snapshots](https://jestjs.io/docs/en/snapshot-testing)

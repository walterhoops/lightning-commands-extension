# Lightning Commands Design Doc

## Product PRD
The Lightning Commands extension is a solution that brings [Commands.dev](https://www.commands.dev/) to Visual Studio Code, enabling searchable and customizable workflows found in the [Warp Terminal](https://www.warp.dev/) to exist within VSCode’s command palette. 

Refer to [PRD](https://docs.google.com/document/d/11PrZeXzZysMXf6-vma_lVyPh81vlWDuL/edit?usp=sharing&ouid=118355910693525651774&rtpof=true&sd=true).

## Requirements
1. The extension should enable the user to search for a workflow in the command palette.
2. The extension should enable the user to generate commands with customized parameter values.
3. The extension should automatically copy the result command to the user's clipboard for pasting in the terminal.

## Proposed Solution
The extension consists of two main components. The Github Action workflow that is responsible for polling and registering the workflow in the extension, and the extension API that takes user inputs and compile the result command to write to the clipboard.

### Github Action
The Github Action workflow runs on a predefined schedule to poll the open-source [workflow library repository](https://github.com/warpdotdev/workflows/tree/main/specs) and downloads all new command workflows to the `resources/workflows` folder. 

Next, to create a user-facing command in VSCode, there are two file changes we need to make.

1. Binding the command ID to a handler function in the extension by calling`vscode.commands.registerCommand`:
```ts=
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {
  const command = 'myExtension.sayHello';

  const commandHandler = (name: string = 'world') => {
    console.log(`Hello ${name}!!!`);
  };

  context.subscriptions.push(vscode.commands.registerCommand(command, commandHandler));
}
```

2. Exposing this command in the Command Palette so it is discoverable by users. The command needs be added to the `contribution` and the `activationEvents` in `package.json`:
```json=
{
  "activationEvents": ["onCommand:myExtension.sayHello"],
  "contributes": {
    "commands": [
      {
        "command": "myExtension.sayHello",
        "title": "Say Hello"
      }
    ]
  }
}
```

Upon downloading the new command workflows from Wrap's public repository, the Github Action workflow should also update corresponding files to bind and expose the command.

### Extension API
#### MultiStepInput Wizard Wrapper
We want to create a wrapper around this [MultiStepInput  Wizard sample](https://github.com/Microsoft/vscode-extension-samples/blob/main/quickinput-sample/src/multiStepInput.ts) provided by VSCode. The current wizard utilizes input function chaining to prompt users to enter multiple inputs. See example below:
```ts=
async function pickResourceGroup(input: MultiStepInput, state: Partial<State>) {
  # implementation details omitted
  return (input: MultiStepInput) => inputName(input, state);
}

async function inputName(input: MultiStepInput, state: Partial<State>) {
  # implementation details omitted
  return (input: MultiStepInput) => pickRuntime(input, state);
}

async function pickRuntime(input: MultiStepInput, state: Partial<State>) {
  # implementation details omitted
}

async function collectInputs() {
  const state = {} as Partial<State>;
  await MultiStepInput.run(input => pickResourceGroup(input, state));
  return state as State;
	}
```

As we can see, the current input chaining is hardcoded as the functions are called out explicitly. We can improve this making a MultiStepInput Wizard wrapper that is shared between all commands. See example below:

```ts=
const args = config.arguments;
let functions = new Array<any>(args.length);

for (let i = args.length - 1; i >= 0; i--) {
  let argName = args[i].name;
  if (i === args.length - 1) {
    // assign function[i] to corresponding async function
  } else {
    // assign function[i] to corresponding async function
    return (input: MultiStepInput) => functions[i + 1](input, state);
}
```

The API should also compile the result command based on the inputs collected and write to the user's clipboard by calling `await vscode.env.clipboard.writeText(text);`

## Demo
[Link to the POC demo](https://drive.google.com/file/d/1MAEOitbSO6XW8rQzwcqaib2bWkm5xYUM/view?usp=sharing)

## Future Work
Some strech features we can work on after the initial release includes:
1. Grouping the workflows based on category.
2. Implement a button that automatically runs the command in user's terminal (no pasting action required).

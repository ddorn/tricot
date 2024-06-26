# TRICOTS: Trace Interception and Collection Tool for Supervision

TRICOTS is meant to collect traces of LLM-agents and scaffoldings and to alter their behavior without needing to modify nor understand the codebase of the agent.

TRICOTS is a simple single-file tool to monitor and edit the messages sent to the OpenAI API using [monkey-patching](https://en.wikipedia.org/wiki/Monkey_patch).
(a technique used to dynamically update the behavior of a piece of code at run-time, without altering the source code)

TRICOTS was developed to enable monitoring as part of [BELLS: Benchmarks for the Evaluation of LLM Supervisors.](https://github.com/CentreSecuriteIA/BELLS)

## Installation

You can install and use TRICOTS in one of two ways in a python 3.9+ environment:
- Copy the [`tricots.py`](./src/tricots.py) file into your project and import it. That's it.
- OR Install it using pip: `pip install tricots`

## Usage

This single file module is a monkey-patch for the OpenAI API that enables:
- Logging of all calls to the OpenAI API
- Modification of the messages sent to the API before sending them

TRICOTS contains two main functions:
- `patch_openai(edit_call)`: Patches the OpenAI API to add logging, with an optional function to edit the list of messages before sending them.
- `new_log_file(path)`: Sets the log file to use, should be before each individual run of the LLM-app to monitor.

### Example: Logging multiple runs of an agent while modifying the system prompt

```python
import tricots

agent = ...
tasks = ["task1", "task2", "task3"]

def edit_call(messages: list[dict]):
    # 1. We can do anything we want with the messages here, and it's fine to modify the list in place.
    # For example we can modify the system prompt to ask the model to always answer in French:
    messages[0]['content'] += "\n\nImportant: always answer in French."
    return messages

# 2. We monkey-patch the OpenAI API (= modify on the fly the libraries of OpenAI library)
# This allows to edit the messages before sending them and once a log file is set, logging them to a file.
tricots.patch_openai(edit_call)

for task in tasks:
    # 3. We set the log file to use for this task.
    tricots.new_log_file(f"logs/{task}.log")
    # 4. We run the agent as usual. Somewhere in its code, it will call the (now patched) OpenAI API.
    agent.run(task)
```

This example shows the main features of TRICOTS:
1. Any user defined `edit_call` function can modify the messages before sending them, if needed. This function takes a list of messages as input and should return a list of messages. Each message is a dictionary, with the same keys as expected by OpenAI API ([see reference here])[https://platform.openai.com/docs/api-reference/chat/create]
2. The `patch_openai` needs to be called at least once to enables logging, with an optional `edit_call` function. `path_openai` can be called multiple times, especially if the `edit_call` function needs to change.
3. The `new_log_file` function should be called at least once, and before each independent run of the LLM-app/agent. It tells TRICOTS where to start logging the (possibly edited) messages. If not called, TRICOTS will only edit the messages.
4. The agent can be run as usual, without any modification to its codebase.

Running this script with an otherwise defined agent will create three log files, `logs/task1.log`, `logs/task2.log`, and `logs/task3.log`, with the messages sent to the OpenAI API, modified to ask the model to always answer in French.

### Structure of the log files

The log files are in the [jsonlines](http://jsonlines.org/) format, with one JSON-encoded API call per line. Each API call is a dictionary with the following structure:
```python
# One line of the log file = one API call
{
    "timestamp": 1713262950.9124262,
    "messages": [
        {
            "role": "system or user or assistant",
            "content": "content of the first message",
        },
        ...
    ]
}
```

The last message in the `messages` field will always be the answer given by the model, so that `messages[:-1]` are messages sent to the API, and `messages[-1]` is the output of the API.

#### Example of a log file

We provide a simple example of a log file, assuming TRICOTS was used to monitor a simple chat app, in which the conversation was:
- System: "Always speak in French."
- Assistant: "Bonjour!"
- User: "What is the CeSIA?"
- Assistant: "Le CeSIA est le Centre pour la Sécurité de l'IA."
- ...

```json
{"timestamp": 10.0, "messages": [{"role": "system", "content": "Always answer in French."}, {"role": "assistant", "content": "Bonjour!"}]}
{"timestamp": 20.0, "messages": [{"role": "system", "content": "Always answer in French."}, {"role": "assistant", "content": "Bonjour!"}, {"role": "user", "content": "What is the CeSIA?"}, {"role": "assistant", "content": "Le CeSIA est le Centre pour la Sécurité de l'IA."}]}
```

There are two lines, because there are two API calls needed to for the two responses of the Assistant. The `"messages"` field contains the messages as sent to the API, with the different keys as specified by [the OpenAI API](https://platform.openai.com/docs/api-reference/chat/create#chat-create-messages)
The last message of a line is always the output of the API, which is the answer of the model.
Furthermore, in basic chat apps, all the previous messages are always sent to the API so that the LLM has the context of the conversation, which is why the first two messages are repeated here.

## Limitations

TRICOTS is a simple tool with a few limitations, but should be easy to extend to fit your needs:
- It works only with the OpenAI API (albeit all versions of it)
- It requires the agent to be written in Python and that it is possible to add TRICOTS to the runtime.
- It requires that the agent can be imported and run as a function call, i.e. it would not work with agents that can only be run from the command line...
    - ... in this case, TRICOTS can still be used by editing the code of the agent to call the patching function, anytime before running.

## Future work

Future improvement to TRICOTS might include:
- Support for other APIs
- Support for changing the model used by the agent (e.g. to use Anthropic API on codebase written only for OpenAI API)
- Interactive visualization of the logs
- Interactive editing of the logs, to manually inspect and modify the messages sent to the API

Those features might be implemented as we need them for the development of BELLS.

## License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE) file for details.

TRICOTS was developed by Diego Dorn, as part of the [BELLS project](https://github.com/CentreSecuriteIA/BELLS), for the [CeSIA — Centre pour la Sécurité de l'IA](https://www.securite-ia.fr/).

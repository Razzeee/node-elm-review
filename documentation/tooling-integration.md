# Tooling integration

This is information dedicated to people who want to use the `elm-review` CLI in a different
environment than a user's terminal, like editors or GitHub actions.

## Things I expect you to do

### Namespacing

If it is possible for your tool to run at the same time as an unrelated `elm-review` CLI run (e.g. an editor), then I'd like you to spawn the CLI using the `--namespace <unique-name-for-your-tool>` (please don't make it look like a semantic version number, to facilitate removing this later on if it happens to be a bad idea).

The CLI creates a bunch of cache inside `elm-stuff/generated-code/jfmengels/elm-review/<namespace>/<CLI version>/` with `cli` as the default `namespace`, including
- `file-cache/`: Caching of the file's ASTs.
- `review-applications/`: Caching of the project's configuration. This is the application we build by compiling the source code in the CLI's `template/` directory.
- `dependencies-cache/`: Caching of the dependencies of the project's configuration computed by `elm-json`. `elm-json` is a bit slow, and doesn't work great offline. This is done so we don't have to compute the dependencies again if the configuration changed but not `review/elm.json`.

Namespacing things means that data will unfortunately be duplicated, but it is meant to prevent different tools from stepping on each other's toes by saving the same files at the same time as another, potentially corrupting the files.

### Provide the path to the Elm compiler and to elm-format

If you happen to know the path to these two executables, please run using
  - `--compiler <path-to-elm>`
  - `--elm-format-path <path-to-elm-format>`. `elm-format` is used to re-format files after they have been fixed (`--fix` or `--fix-all`). If you are not running that, you have no need to specify `--elm-format-path`. As you will see in a different section below, the JSON format will give you the necessary steps to perform a fix yourself.

This will help the CLI by not trying several paths before succeeding. If you don't specify these, this is how the binaries are found:
  - `elm`: We run `which elm` and use that path.
  - `elm-format`: We run `npx --no-install elm-format`, and fallback to a `elm-format`.

If you are unsure that these paths are correct, then maybe omit these, as the CLI
will not attempt to fallback to other paths.

Note: At the moment, we are reformatting a file with `elm-format` **after** is has been saved to the file system. This is not ideal, and I'd love help with improving that.

## Format of the JSON

If you desire to get the output of the CLI as JSON, you can run with `--report=json`. The output will look like what the following sections describe.

### Review errors

If the process ran without any hitches, you should get something like the following:

```json
{
  "type": "review-errors",
  "errors": [
    {
      "path": "src/Review/Exceptions.elm",
      "errors": [
        {
          "rule": "NoUnused.Variables",
          "message": "Top-level variable `fjoziejf` is not used",
          "details": [
            "You should either use this value somewhere, or remove it at the location I pointed at."
          ],
          "region": {
            "start": {
              "line": 19,
              "column": 1
            },
            "end": {
              "line": 19,
              "column": 9
            }
          },
          "fix": [
            {
              "range": {
                "start": {
                  "line": 19,
                  "column": 1
                },
                "end": {
                  "line": 20,
                  "column": 1
                }
              },
              "str": ""
            }
          ],
          "formatted": [
            {
              "str": "(fix) ",
              "color": [
                51,
                187,
                200
              ]
            },
            {
              "str": "NoUnused.Variables",
              "color": [
                255,
                0,
                0
              ]
            },
            ": Top-level variable `fjoziejf` is not used\n\n19| fjoziejf=1\n    ",
            {
              "str": "^^^^^^^^",
              "color": [
                255,
                0,
                0
              ]
            },
            "\n20| type Exceptions\n\n\nYou should either use this value somewhere, or remove it at the location I pointed at."
          ]
        }
      ]
    }
  ]
}
```

- `type`: Equal to `"review-errors"` when the run went well (finding errors or not)
- `errors`: The array of errors that `elm-review` found. If it is empty, then no errors were found (in a normal run, `elm-review` would then exit with status code 0). The following describe each item in the array.
  - `path`: The relative path to the file for which the (sibling) errors are reported.
  - `errors`: The array of errors that `elm-review` found for this file. The following describe each item in the array.
    - `rule`: The name of the rule that reported this error.
    - `message`: A short description of the error. If you show a summary of the errors, this is what you will want to show, along with `rule`.
    - `details`: A longer description, providing more details about the error and often a description of how to resolve it. Every string in this array of strings corresponds to a paragraph.
    - `region`: The region in which this error occurred. The `line` and `column` values start from `1`, not `0`.
    - `fix` (optional): A list of fixes/edits to automatically solve the errors. Each "edit" describes a range (1-based) in the source code to replace, and what to replace it by. If `str` is empty, it means we are removing code, if the `start` and `end` are the same, we are inserting code, otherwise we are modifying code.
    In the CLI, these are applied one-by-one, starting from the ones that are near the end of the file. When applying them, the CLI makes sure that there are no overlapping ranges and that the fix results in an Elm file without syntax errors. These are all steps that you need to do yourself at the moment.
    (Proposal to be discussed: maybe the CLI can be spawned with this fix data and apply its own algorithm, to avoid you having to do all this work?)
    - `formatted`: An array of strings and objects that represent the full human-readable error that would be shown to the user. The simple strings have no special formatting, and the objects have a `color` of the form `[<0-255>, <0-255>, <0-255>]`. In future releases, more formatting options like `bold` could be added, but they are not here at this moment.


### CLI errors

Everything doesn't always go as planned, and sometimes we run into problems we anticipated and others that we didn't.
In that case, we (should) still report errors as JSON, with the following format:

```json
{
  "type": "error",
  "title": "COULD NOT FIND ELM.JSON",
  "path": "elm.json",
  "message": "I was expecting to find an elm.json file in the current directory or one of its parents, but I did not find one.\n\nIf you wish to run elm-review from outside your project,\ntry re-running it with --elmjson <path-to-elm.json>.",
  "stack": "Error: I was expecting to find an elm.json file in the current directory or one of its parents, but I did not find one.\n\nIf you wish to run elm-review from outside your project,\ntry re-running it with --elmjson <path-to-elm.json>.\n    at Object.projectToReview (/home/jeroen/dev/node-elm-review/lib/options.js:46:13)\n    at Object.build (/home/jeroen/dev/node-elm-review/lib/build.js:42:35)\n    at runElmReview (/home/jeroen/dev/node-elm-review/lib/main.js:62:41)\n    at module.exports (/home/jeroen/dev/node-elm-review/lib/main.js:105:3)\n    at Object.<anonymous> (/home/jeroen/dev/node-elm-review/bin/elm-review:3:23)\n    at Module._compile (internal/modules/cjs/loader.js:1144:30)\n    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1164:10)\n    at Module.load (internal/modules/cjs/loader.js:993:32)\n    at Function.Module._load (internal/modules/cjs/loader.js:892:14)\n    at Function.executeUserEntryPoint [as runMain] (internal/modules/run_main.js:71:12)"
}
```

- `type`: Equal to `"error"` when things are unexpected
- `path`: The relative path to a file we could trace the problem to, or to a default one. This is in a lot of cases using a default value, because we are not able to pinpoint to a specific file. Also the default file might not exist, which may be the cause of the error.
- `message`: The helpful description of the problem. This is meant for humans to read, but colors have been removed and it has been trimmed. You may wish to remove the line-breaks maybe? In the future, it may become an array like the `formatted` message for review errors.
- `stack`: The original JavaScript runtime stacktrace. Only sent if you run with `--debug`.

## Things that may help you

Running with `--debug` will:
- Add the stack trace when you run into an (un)expected error while running the CLI
- Pretty print the JSON output, and add the stack trace to it.

If you run into trouble, you can safely delete the `elm-stuff/generated-code/jfmengels/elm-review` directory.

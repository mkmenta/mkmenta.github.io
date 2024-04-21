---
layout: post
title:  "VSCode for AI development in Python (and more)"
date:   2024-01-21 12:00:08 +0100
categories: vscode ai-workflow
---
I will summarize in this post some of the most important settings I use in my everyday development of Artificial Intelligence (AI) in Python. 
This setup is not exclusive for AI development as many settings are valid for other development areas and programming languages. 
However, I find this setup very useful for my AI development work.

I also need to say that I come from using PyCharm Professional, so you might find this post at hand if you are considering migrating to VSCode.

## 1. Remote development
In AI development, many times we need to execute code in powerful machines with GPUs that are located remotely. 
VSCode handles remote development very well: it allows you to edit, debug, and run code directly in the remote machines as if you were working 
locally.

How to setup:
1. Set the `~/.ssh/config` with your remote machine values. 
This BTW allows you to connect to machines just by writing `ssh NAME_TO_IDENTIFY_THE_REMOTE_MACHINE` in the terminal (you can find more info online about it). 
An example:
```
Host NAME_TO_IDENTIFY_THE_REMOTE_MACHINE1
    HostName REMOTE_MACHINE_USERNAME
    User REMOTE_MACHINE_USERNAME
    Port REMOTE_MACHINE_SSH_PORT
    IdentityFile LOCAL_PATH_TO_PUBLIC_KEY
    # Some other config you might want to add:
    ServerAliveInterval 240
    ServerAliveCountMax 2
    LocalForward 80 127.0.0.1:80
Host NAME_TO_IDENTIFY_THE_REMOTE_MACHINE2
    ...
```
2. Open VSCode and install the "Remote Development" extension (Extension ID: `ms-vscode-remote.vscode-remote-extensionpack`).
3. Click on the bottom left corner "><" button, then "Connect to host..." and finally select the machine in which you want to code.
Now the whole VSCode will work directly in the remote machine including: the project and files explorer, the terminal, and the debugger.

The drawback of this is that you won't have the files on your local machine, so make sure to push all changes in case you lose access to the remote host.

If you come from PyCharm:
```
In PyCharm you change your files locally and then upload them to the remote host in order to execute them. In VSCode you directly edit the files in the remote host.
```
A useful tip for moving around or opening files outside the project in remote hosts (instead of clicking on File --> Open...) is to use the command `code FILE_OR_FOLDER` in the VSCode terminal.

## The settings
There are three main levels of settings:
- User: apply globally to all VS Code instances.
- Remote: apply to a remote machine specifically for a user.
- Workspace: apply to the open folder or workspace/project. 

Some things:
- The order of preference/override is Workspace > Remote > User.
- You can find the JSON containing the changed settings by clicking: View -> "Command Palette..." -> "Preferences: Open XXX settings (JSON)".
- You can look for the Setting IDs that I link in this post directly by searching them in the bar of the "Settings" UI or adding them to the JSON. Example of ID: `python.defaultInterpreterPath`.

## The Python interpreter
Once you open a `.py` file (and wait for the Python extension to install) you will be able to select a Python interpreter in the bottom-right corner. This means that once you open the project it will activate that virtual environment (venv) in every new terminal and use that environment for the debugger. 

Some things:
- To select a venv's interpreter, select the following binary: `PATH_TO_THE_VENV/bin/python`.
- You can also set a "Default Interpreter Path" in your settings (under Extensions/Python). Setting ID: `python.defaultInterpreterPath`.
- If you need to add paths so that they are recognized in the editor and automatically added to the `PYTHONPATH` edit the following settings: `python.analysis.extraPaths`, `python.analysis.extraPaths`.

## Customize the user interface
You can hide parts of the user interface (UI) that you don't use to have a cleaner view. 
- Right-click on the left panel icons to hide/show them.
- Click on the `...` at the top-right corner of the left panel to show/hide sub-panels.
- Click on the three icons at the top-right of VSCode to show/hide the panels at any moment.

## The debugger
Initialize the debugger configurations by opening a `.py` file, clicking in the "Run and Debug" tab on the left and then clicking in "create a launch.json" file.

The `launch.json` configuration structure:
```json
...
{
    // Equivalent to `python3 train.py --num-workers=0 --transforms flip "random crop"`
    "name": "Train model",
    "type": "debugpy",
    "request": "launch",
    "program": "train.py",
    "console": "integratedTerminal",
    // If justMyCode=true, it won't allow you to debug/stop in libraries external to the current workspace
    "justMyCode": false,  
    "env": {
        "CUDA_VISIBLE_DEVICES": "0",
        // ${workspaceFolder} is the current project's folder
        "PYTHONPATH": "$PYTHONPATH:${workspaceFolder}:${workspaceFolder}/../my-other-lib"
    },
     "args": [
        "--num-workers=0",
        "--transforms",
        "flip",
        "random crop",
    ]
},
...
```
for debugging commands like `streamlit run demo.py --server.port=1234` use `module` instead of `program`:
```json
{
    "name": "Demo in streamlit",
    "type": "debugpy",
    "request": "launch",
    "module": "streamlit",
    "console": "integratedTerminal",
    "justMyCode": true,
    "args": [
        "run",
        "demo.py",
        "--server.port=1234"
    ],
},
```

## Automatic formatting
You will need to have the `autopep8` extension installed (ID: `ms-python.autopep8`) and have the following settings:
```json
{
...
    "python.formatting.provider": "autopep8",
    "autopep8.args": [
        "--ignore=E402", // check these here: https://pycodestyle.pycqa.org/en/latest/intro.html#error-codes
        "--max-line-length=140",
        "--experimental" // otherwise --max-line-length does not work
    ],
    "editor.formatOnSave": true,
    "editor.formatOnPaste": true,
...
}
```
I would set up those settings at the User level and overwrite them at the Workspace level if necessary.

Tip: if you disable it by default, you can just format the selected lines by right-clicking on top and clicking on "Format Selection".

## Automatic import sorting
You will need to have the `isort` extension installed (ID: `ms-python.isort`) and have the following settings:
```json
{
...
    "[python]": {
        "editor.codeActionsOnSave": {
            "source.organizeImports": "explicit"
        },
      },
...
}
```
I would set up those settings at the User level and overwrite them at the Workspace level if necessary.

Tip: if you disable it by default, you can just organize the imports in the current file with `Shift + Alt + O`.

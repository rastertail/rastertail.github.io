+++
title = "Hacking VS Code for On-the-Fly Environment Switching"
date = 2022-03-27
+++

**Note: This technique no longer works as of VS Code 1.66, presumably due to the upgrade to Electron 17! I intend to find a new solution, but it is not planned any time soon.**

In my day-to-day development workflows, I make heavy use of [Nix Flakes](https://nixos.wiki/wiki/Flakes); particularly, the development shell feature where I can make a special, reproducable development environment for each of my projects.
To enter a flake environment, you use the `nix develop` command, which interacts great with terminal editors like Vim.
However, I have recently switched over to using VS Code for my heavy-duty programming, and launching it from a terminal after running `nix develop` is not the smoothest workflow.
In order to comfortably use VS Code every day, I had to find a better solution.

<!-- more -->

Existing VS Code extensions like Roman Valihura's [Nix Environment Selector](https://github.com/arrterian/nix-env-selector) let you switch into environments defined by Nix shells without flakes, but nothing had been developed for flakes... until now!
I've been working on an extension, [Nix Flake Tools](https://github.com/rastertail/nix-flake-tools), which provides functionality to *fully replace* VS Code's environment with that of a Nix flake development shell, something never seen before.
To achieve this, though, I pulled off some incredibly disgusting hacks, which ultimately are the motivation for this article.

### Prior Art: Nix Environment Selector

Of course, the obvious solution to this environment-switching conundrum would be to simply extend Nix Environment Selector to support flake-based development shells.
However, I decided the prospect of an extension with deeper flake integrations than just development shells would be more interesting, so I started looking into developing my own extension, potentially borrowing some techniques from Nix Environment Selector.
So, let's take a look at how Nix Environment Selector goes about setting environment variables:

```clojure
(defn set-current-env [env-vars]
  (mapv (fn [[name value]]
          (aset js/process.env name value))
        env-vars))
```

Despite being written in ClojureScript, it's pretty easy to tell what is going on here: The extension simply overwrites values in `process.env`, which in turn sets environment variables on VS Code's extension host process.
In theory, this makes sense, as much of VS Code's functionality exists as internal extensions, so setting environment variables on that should set the environment for much of the application.
In practice, though, it... does not work.
Admittedly, I never actually tried using Nix Environment Selector, so maybe it works, and I'm missing a key step, so if that's the case, let me know!
In my own extension, though, I found that setting values on `process.env` had no immediate effect on extensions like rust-analyzer, and reloading the workspace would end up restarting the extension host, resetting all of my changes.

Either way, this approach comes with some drawbacks.
Namely, it definitely [doesn't influence the built-in terminal](https://github.com/arrterian/nix-env-selector/issues/31), nor does it [work in build tasks](https://github.com/arrterian/nix-env-selector/issues/55).
Personally, this doesn't quite cut it for my workflows, so I had to start looking for other solutions.

### The Blessed Solution

Luckily, I'm not the only person in the world who wants to influence the environment VS Code runs under on the fly.
Thus, after [rejecting modifying extension environments](https://github.com/microsoft/vscode/issues/46696#issuecomment-558733819), the VS Code team implemented an `EnvironmentVariableCollection` API, which allows changing the environment of the built-in terminal, as well as certain task types.
In my experience, this worked for what it is designed to do, but for my workflows, that just was not enough.
Critically, since I restrict most language toolchains to Nix development environments, extensions need to see the modified `PATH` to pick up toolchain executables.
Theoretically, combining this API with the previous approach could cover most of my use cases, but again, I could not get `process.env` to actually achieve anything useful, so I had to keep searching.

### A Crime Against Software

At this point, I was desparately searching StackOverflow for answers on how to modify the environment of a running process.
Finally, I came across [an answer](https://stackoverflow.com/a/211064) which suggested attaching to a process with `gdb` and calling `putenv`.
To quote the author of the reposnse, Andrew, *This is quite a nasty hack and should only be done in the context of a debugging scenario, of course.*
That got me thinking though... does NodeJS have a debugger I can attach to VS Code?

...Yes. It does.

I tried for myself, running `node inspect` with the PID of VS Code's root process, entered a REPL, and ran `process.env["PATH"] = "lol";`.
Nervously, I reloaded the workspace in my VS Code instance, and just as I hoped, the `PATH` variable was successfully overwritten.
Andrew's words echoed in the back of my mind... but my laughter drowned them out.
So, how can we do this programatically?

After digging around a bit, this ended up being *extremely* easy.
To make a NodeJS process open to debugger connections, you simply send it `SIGUSR1`, or even easier, call `process._debugProcess(pid)` from another NodeJS process.
Particularly, this opens a WebSocket server on port 9229 following the [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/).
The protocol is simple enough that I could have written a client myself, but of course, there already exists [an NPM package](https://www.npmjs.com/package/chrome-remote-interface) to do that for me.
Anyway, VS Code makes this part extremely easy, as it already exports a `VSCODE_PID` environment variable, so I can just attach a debugger to that.
At this point, all that remains is sending the right code over via the `Runtime.evaluate` command.
The final code looks like this:

```typescript
export async function injectEnvironment(vars: Map<string, string>) {
    // Get root process from environment (thanks!)
    const rootProc = parseInt(process.env["VSCODE_PID"]!);

    // Start debug session on the root process
    (process as any)._debugProcess(rootProc);

    // Connect debugger
    const dbg = cdp({ host: "127.0.0.1", port: 9229 });

    // Apply environment variables
    for (const [name, value] of vars) {
        // Escape quotes and backslashes in variable values
        value.replace(/\\/g, "\\\\");
        value.replace(/"/g, '\\"');

        await dbg.Runtime.evaluate({
            expression: `process.env["${name}"] = "${value}";`,
        });
    }

    // Close debugger
    await dbg.close();
}
```

This achieves everything I want, successfully replacing the environment of the entire VS Code application after a workspace reload.
In practice, I also inject some extra code to save the old environment so the user can restore it later, which also ends up a bit unusual, saving the copy of the environment as a global variable on the VS Code root process.
Despite my success, though, this easily passes as the most horrific code I have ever written thus far, and I sincerely hope that nothing else surpasses it.

The extension implementing this is available [here](https://github.com/rastertail/nix-flake-tools), though I hope to never publish it on the extension marketplace unless I can find a better way to approach the problem.
You can still build the VSIX with `nix build github:rastertail/nix-flake-tools#vsix`, and the extension is also provided under the default `extension` package in a NixOS-friendly derivation.

If you choose to use this in your own workflow, please remember Andrew's words.

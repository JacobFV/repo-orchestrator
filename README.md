![](/assets/logo/logo.png)

# Repo Orchestrator

  1. From the bottom repo's and upwards
    1. Make appropriate git changes
        1. Determine desired git properties
        2. Determine current state
        3. Generate change plan
        4. Execute (deferred or immediately)
      2. Make appropriate code organizational changes
        1. Determine desired code organization
        2. Determine current state
        3. Generate change plan (this may be hard for full-on python objects. no it's not)
        4. Execute (deferred or immediately)



I need to make a generic traversal program. Then different programs can be run on each subrpeo like git commit, git push, submodularize, align, etc.

Repo Orchestrator is a repository structuring and organization tool providing the following primary features:

- `frame`: builds nested-repo skeletons or modifies existing repo's from a config
- `align`: add/modify files to align structure of existing repository to a specified template
- `commit-and-push`: commit and push changes recursively

## Getting Started

Repo Orchestrator is available as an `npx` script so you don't have to download anything to use it. Simply run:

```bash
npx jacobfv/repo-orchestrator <command> <args>...
```

## Usage

Note: Please use this tool at in same working directory as your repository. 

### Frame

`npx jacobfv/repo-orchestrator frame [-y] [REPO_PATH] [--config CONFIG]`

`frame` builds a nested repository skeleton from the config file or modifies the existing repository passed in the CLI or specified in the config file or both. 
The main algorithm is as follows:

```
Algorithm: Main
----------------
Parse config file if present
  Determine file type (json, yaml, toml) and open it
  Convert to JSON
  Replace any template variables
  Update default config object with this config
Update main repo is REPO_PATH is supplied
Call `Core(confirm=y, config=config_obj)`
Save the clarified repo structure to output file

Algorithm: Core(confirm: boolean, config: object)
----------------
For each subdir in post-order DFS (leaves to root) in the file system:
  If that subdir should be / is a subrepo:
    If not already a git repo:
      maybe confirm
      git init
    For all subrepos underneath it in the config tree:
      maybe confirm
      Delete them
      Add them as submodules
    Stage and commit all changes
    If remote doesn't already exist:
      If `createRemotes` is false:
        Throw error
      maybe confirm
      Create remote
    maybe confirm
    push changes to remote
    add subrepo to config tree

Algorithm: If that subdir should be / is a subrepo
----------------
If the object in the config tree at this path (relative to main) has a `submodule=True` attribute or any of the other submodule fields:
  return True
For inode in all filesystem objects below subdir:
  If inodes.glob matches any of the `submodularizeIfHas` globs:
    return True
return False
```

Format:
```ts
// This is the default format structure. All parameters are optional.
// It can be overriden by passing in a config file in the CLI.

vars: {[key: string]: string} // declare as key/value pairs; reference with `$key`
submodularizeIfHas: string[] = ['.git/*', '.gitignore', '.gitmodules', 'README.md']
remote: {
  provider: "github" = "github", // leaves oppertunity for adding other providers
  account: string|null = null, // defaults to current user
  createRemotes: boolean = true, // create remote repo's for new submodules
}
inferStructure: boolean = true // whether to infer / modify `main` repository structure if (entirely or partially) unspecified. If `false`, ignores subdirs with README's or .git subdirs. (Doesn't submodularize them.)
main: RepoConfig | string = '.' // specify path to repo root if `string`
```

```ts
// type definitions
type DirConfig {
  dirs: {[key: string]: DirConfig}
  output_notes: string
}

type RepoConfig = DirConfig & {
  name: string, // defaults to name of containing directory
  url: string, // defaults to `${remote_url_base}/${remote.account}/${name}`
  subrepo?: boolean, // used to explicitly indicate that this is a subrepo, not just a directory
  ...alignArgs: string // defaults to global `alginArgs`; used to pass in arguments to `align`
}
```

Here's an example config file:

```yaml
vars:
- custom_template: "./custom-template"
- custom_template2: "https://github.com/xyz/abc"

main:
  git: True
  dirs:
    - subdir1:
      - subrepo1.1:
        subrepo: True  # only needed if cannot be inferred from `submodularizeIfHas` (assumed false otherwise)
        template: custom_template
        dirs:
          ...
      - subrepo1.2:
    - subrepo2:
      dirs:
        ...
```

### Algin

`npx jacobfv/repo-orchestrator align <template-repo> [[--substition / --no-substituion] [--url https://github.com/abc/xyz] [--name xyZ]]`

`align` is like a `diff` and `git merge` with substition features. The `<template-repo>` argument can be a git url or a relative or absolute path to a local directory. The main algorithm is as follows:
- run a `diff` between the template and main repo's
- add all files in template that are not in main
- for modified files,
  - If a line is present in main and is highly similar to one in template, then keep the line in main but include the template line directly below it with a `>>> ` prefix.
  - If a line is present in main and is not highly similar to one in template, keep it.
  - If a line is absent in main but present in template, add it.
  Then developers are responsible for manually searching through the repository to handle lines with a `>>> ` prefix.
- If `--no-substituion` is *not* supplied, replace `{{TEMPLATE}}` tokens with a value based on the repository metadata. (Substitution is default but can be explicitly set with the `--substitution` flag.) The following template statements are supported:
  - `{{URL}}`: URL of repository remote. Defaults to primary pull remote url if not supplied in `--url` arg.
  - `{{NAME}}`: Name of repository. Defaults to final path element of url if not supplied in `--name` arg.
  - other cli arguments are treated as if they are defining custom template tokens.

*Implement architecture-level template renderign in a separate tool.*

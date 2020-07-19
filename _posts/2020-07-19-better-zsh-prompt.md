---
title: A better zsh prompt for python development
author: Emerson Harkin
---

I switched to zsh recently and there's a lot to like. The [agnoster
theme](https://github.com/agnoster/agnoster-zsh-theme) from [Oh My
Zsh!](https://ohmyz.sh/qp) provides a lot more useful information than I'm used
to having in my bash prompt (the current git branch, and the exit status of the
last command, for starters), and it just works out of the box.

![Agnoster looks great out of the box](/assets/img/agnoster/zsh_agnoster_stock.png)

Undeterred by agnoster's excellent defaults, I spent some time tweaking the
prompt and came up with something that works just a little bit better for
python development. My version is Anaconda-aware, and displays a shorter path
when inside a git repo.

![Agnoster for pythonistas](/assets/img/agnoster/zsh_agnoster_modded.png)

## Minor gripes with agnoster

### Gripe #1: It isn't Anaconda aware

I'm usually working on at least a couple of different python projects at any
one time, each with its own Anaconda virtual environment. Agnoster can display
the name of the virtual environment that you're currently in as part of your
prompt, but unfortunately it doesn't detect Anaconda virtual environments out
of the box. Even worse, Anaconda prepends the name of the current environment
to the beginning of your prompt by default, spoiling Agnoster's powerline
aesthetic.

#### Solution

If you want something done right, you have to do it yourself. First, stop
Anaconda from modifying your prompt by adding the following line to your top
level `.condarc`.
```yaml
changeps1: false
```
(If python isn't literally the only thing you use in the terminal, you may also
want to stop Anaconda from activating the base environment automatically. This
can be configured by adding `auto_activate_base: false`.)

Next, we need to add the name of the active Anaconda environment to the prompt.
The `prompt_virtualenv()` shell function controls how the virtual environment
part of the agnoster prompt is displayed, so we just have to add a couple lines
to insert the name of the current Anaconda environment (if one is active). The
default `prompt_virtualenv()` looks like this.
```sh
prompt_virtualenv() {
  if [[ -n $VIRTUAL_ENV ]]; then
    color=cyan
    prompt_segment $color $PRIMARY_FG
    print -Pn " $(basename $VIRTUAL_ENV) "
  fi
}
```
We can get the name of the current Anaconda environment from the `CONDA_PREFIX`
environment variable.
```sh
$ conda activate whatever
$ echo $CONDA_PREFIX
/opt/miniconda3/envs/whatever
```
Since this actually gives the absolute path to the Anaconda environment, we can
extract the name of the environent with `basename $CONDA_PREFIX`. Adding this
into `prompt_virtualenv()` (and applying a touch of refactoring), we get the
following.
```sh
prompt_virtualenv() {
  # Get the name of the virtual environment if one is active
  if [[ -n $VIRTUAL_ENV ]]; then
    local env_label=" $(basename $VIRTUAL_ENV) "
  fi

  # Get the name of the Anaconda environment if one is active
  if [[ -n $CONDA_PREFIX ]]; then
    if [[ -n $env_label ]]; then
      env_label+="+ $(basename $CONDA_PREFIX) "
    else
      local env_label=" $(basename $CONDA_PREFIX) "
    fi
  fi

  # Draw prompt segment if a virtual/conda environment is active
  if [[ -n $env_label ]]; then
    color=cyan
    prompt_segment $color $PRIMARY_FG
    print -Pn $env_label
  fi
}
```

The above gives a prompt that looks something like this (python icon not
included).

![Conda-aware agnoster](/assets/img/agnoster/zsh_conda.png)

### Gripe #2: The prompt is too long

By default, agnoster shows the full path from `/` or `HOME` to your working
directory. I find this to be a bit overkill; if I really want to know the full
path, I'll just use `pwd`. Most of the time, I only care about where I am
relative to the top of a git repo.

#### Solution

Override the `promp_dir()` shell function to display a shorter path. What
follows is a shell function in three acts:

1. Get the path from root, home, or the top of the current git repo (whichever
   is shortest) to the working directory.
2. Shorten directories above the current one to only one letter (for example,
   `path/to/dir` becomes `p/t/dir`).
3. Draw the prompt segment.

```sh
prompt_dir() {
    # Get the path from home, root, or git repo to the working directory
    if [ -d .git ]; then
        # If the current directory is the top level of a git repo,
        # just add the name of the repo to the prompt and exit.
        prompt_segment blue $CURRENT_FG
        print -Pn "$(basename $(pwd)))"
        return 0
    elif $(git rev-parse > /dev/null 2>&1); then
        # If we're in a git repo, get the path from the top of the repo to the
        # working directory.
        local abs_path_=$(pwd)
        local git_toplevel="$(git rev-parse --show-toplevel)"
        local path_=${abs_path_#$git_toplevel}
    else
        # If we aren't in a git repo, get the path from either root or home to
        # the working directory.
        local abs_path_=$(pwd)
        local path_=${abs_path_#$HOME}

        if [[ $abs_path_ != $path_ ]]; then
            local path_="~/$path_"
        else
            local from_root=1
        fi
    fi

    # Shorten the path by truncating each directory (except the current one) to
    # only one letter.
    local path_as_array=(${(s:/:)path_})  # Split the path at forward slashes
    local length_of_path=${#path_as_array[@]}
    local shrunken_path=""
    if [[ $length_of_path != 0 ]]; then
        for i in {1..$length_of_path}; do
            if [[ $i != 1 || $git_toplevel ]]; then
                # Add a separating slash
                shrunken_path+="/"
            fi

            # Truncate the dir name to the first letter, unless it is the
            # current dir
            if [[ $i != $num_path_inds ]]; then
                local elem="$path_as_array[$i]"
                shrunken_path+="${elem[0,1]}"
            else
                local elem="$path_as_array[$i]"
                shrunken_path+="$elem"
            fi
        done
    fi

    if [[ $from_root ]]; then
        local shrunken_path="/"$shrunken_path
    elif [[ $git_toplevel ]]; then
        local shrunken_path="$(basename $git_toplevel)$shrunken_path"
    fi

    # Draw the prompt
    prompt_segment blue $CURRENT_FG
    print -Pn "$shrunken_path"
}
```

This will give you a prompt that looks something like the following (git repo
icon not included).

![Repo-aware agnoster](/assets/img/agnoster/zsh_directory.png)

## Further reading

For the curious, you can find the full versions of my config files
[here](https://github.com/efharkin/dotfiles).

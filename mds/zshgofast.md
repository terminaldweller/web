# How to make the zsh prompt faster

In this post, we are going to take a look at 3 ways of making the zsh prompt return faster.<br/>
We will talk about:

- plugin precompilation
- caching
- async functions

There are obviously other methods, of course, but you can find them on almost any other blog talking about zsh prompts so for the sake of brevity, I'm not going to repeat them(an example: use [fnm](https://github.com/Schniz/fnm) instead of [nvm](https://github.com/nvm-sh/nvm) or why would you not use [mise](https://github.com/jdx/mise)). There is also a faster syntax highlighter for zsh. so on and so forth.<br/>

Also:

- Using the color red for all segments in your zsh prompt will make it go faster because, as everyone should know by now, "red ones go faster"

Before we start though, a bit of reasoning on why I would be doing the things the way I am doing them. A prompt written entirely in a faster language will be faster, yes, but I'm not willing to write a couple of thousand lines of code to get my current shell prompt(the major issue here being maintenance throughout the years,i.e. technical debt). My shell prompt is pretty critical for me so I want to be in complete control over it which means I am unwilling to use ready-made prompts with thousands of lines of code which don't have half the segments that I need.

And finally:

```txt
If you say you care about your zsh prompt's performance then that means that you already have `zmodload zsh/zprof` in your rc file.
```

If you talk about performance, then you have to have benchmarks and profilers. `zprof` is built in.<br/>

## Plugin Precompilation

zsh can compile zsh scripts using the builtin `zcompile` into wordcode. This will have the effect of having faster parsing.
The way we use this to get a faster prompt is to explicitly ask zsh to compile certain chunky plugins(think your [syntax highlighters](https://github.com/zsh-users/zsh-syntax-highlighting) and [completion](https://github.com/zsh-users/zsh-autosuggestions) plugins) into wordcode.

For example:

```zsh
zcompile-many ~/.oh-my-zsh/custom/plugins/zsh-autosuggestions/{zsh-autosuggestions.zsh,src/**/*.zsh}
zcompile-many ~/zsh-async.git/v1.8.5/async.zsh
```

You can read more about `zcompile` in `man 1 zshall`, under `SHELL BUILTIN COMMANDS`, under `zcompile`.

## Caching

### Eval Caching

First we will talk about evalcaching: caching the results of `eval`s. This is a thing because some heavier/older version managers use this to inject themselves into your shell and some of them just take too much time.
For those kinds of plugins you can use [evalcache](https://github.com/mroth/evalcache/).
After sourcing the script, you can use it like so:

```zsh
_evalcache rbenv init -
```

We basically just swap out all instances of `eval` with `_evalcache`.<br/>

Admittedly, this is very limited in scope and it has no smart way of redoing the cache. A function is provided, `_evalcache_clear`, which will clear the cache, which in turn will result in the cache being regenerated.

### Ye Old Result Caching

You can also just cache some results in a file and then just read from a file. Now how you update the cache is really up to you. You can use a user service, you can use cron jobs. You can even run it in the prompt itself or run the function asynchronously which we will get to in the next section.<br/>

Below is a simple example of implementing such a cache:

```bash
caching_function() {
  CACHE_OUTPUT="$1"
  TIME_CACHE=$2

  # check if the cache result is too old
  if [ $(($(stat --format=%Y "$CACHE_OUTPUT") + TIME_CACHE)) -gt "$(date +%s)" ]; then
    # the cache is still valid, we don't need to do anything
    :
  else
    # the cache value is too old
    if OUTPUT=$("your function that generates the output goes here"); then
      if [ -n "${OUTPUT}" ]; then
        echo "${OUTPUT}" >"${CACHE_OUTPUT}"
      fi
    fi
  fi

  cat "${CACHE_OUTPUT}"
}
```

Our function has two arguments:

- The first arg tells it where to store the cache. I keep mine under `/tmp`.
- The second is the amount of time the cache result will be valid for.

The function will check the last time the file was updated and compare that using the `CACHE_TIME` var and the current time. If the cache value is new enough, we just return the cache value and we are done. If the cache is old, we run the function that generates the result, update the cache and return the new result.<br/>

## Async Functions

This is probably the most important one. Again, no surprises here. We will use a zsh async library,[zsh-async](https://github.com/mafredri/zsh-async), to have some async segments in our zsh prompt.

Let's take this zsh function from my prompt:

```zsh
docker_compose_running_pwd() {
  local cwd="$1"
  local list=$(docker compose ls --format json | jq '.[].ConfigFiles' | tr -d '"')
  local array=("${(@f)$(echo ${list})}")
  for elem in "${array[@]}"; do
    if [[ "${cwd}" == $(dirname "${elem}") ]];then
      echo "\U1F7E6"
      return
    else
        ;
    fi
  done
}
```

This function prints a blue square if there is a docker compose file running from the current path we are in and outputs nothing if we are not.<br/>
Under normal circumstances this does not require to be an async segment but as soon as you switch your docker context to a remote docker host, then you will start to feel the pain.<br/>

We first define a start function that registers an async worker with the library and then define a callback function that gets called when the async runner returns:

```zsh
_async_dcpr_start() {
  async_start_worker dcpr_info
  async_register_callback dcpr_info _async_dcpr_info_done
}
```

The start function has nothing special in it. We just register an async worker and then register a callback function that will be called when the runner returns. Please do note that `async_register_callback` will require two arguments, the name of the async runner and the name of our callback function.<br/>
Next we will define our callback function:

```zsh
_async_dcpr_info_done() {
  #first part
  local job=$1
  local return_code=$2
  local stdout=$3
  local more=$6
  #second part
  if [[  $job == '[async]' ]]; then
    if [[ $return_code -eq 2 ]]; then
      _async_dcpr_start
      return
    fi
  fi
  #third part
  dcpr_info_msg=$stdout
  #fourth part
  [[ $more == 1 ]] || set-prompt && zle reset-prompt
}
```

The callback functions gets 6 arguments(copied from [here](https://github.com/mafredri/zsh-async/blob/main/README.md)):

- $1 job name, e.g. the function passed to async_job
- $2 return code
  - Returns -1 if return code is missing, this should never happen, if it does, you have likely run into a bug. Please open a new issue with a detailed description of what you were doing.
- $3 resulting (stdout) output from job execution
- $4 execution time, floating point e.g. 0.0076138973 seconds
- $5 resulting (stderr) error output from job execution
- $6 has next result in buffer (0 = buffer empty, 1 = yes)
  - This means another async job has completed and is pending in the buffer, it's very likely that your callback function will be called a second time (or more) in this execution. It's generally a good idea to e.g. delay prompt updates (zle reset-prompt) until the buffer is empty to prevent strange states in ZLE.

The function itself is straightforward. I like to rename the shell arguments so i have to deal with a name a couple of months from the time of writing the function and not some random numbers.<br/>

Next is the part where we handle the errors. These are the errors returned by the async runner. I like to call the start function of the async job again to get it to start again. This is not, generally speaking, a good idea since if something is broken and a rerun won't fix it, you end up in a loop. This is essentially my version of having the async job "failing loudly".<br/>

The third part is where we assign the stdout that our function, `docker_compose_running_pwd`, made to a "global" variable. This variable,`dcpr_info_msg`, is the variable that we will use in our prompt.<br/>

The fourth part is the most important part. The first condition checks for an empty buffer. If the buffer is not empty, it means we have more async jobs that have finished and are waiting for their turn, in which case it will not update the prompt. If, however the prompt buffer is empty, then we will `set` and `reset` the prompt to get an updated prompt displayed when the buffer is empty.<br/>

This is ideal, because, first and foremost, the library's documentation tells us to do that to avoid having `zle` ending up in a weird state. The other reason would be to avoid unnecessary zle resets to cut down on needless flickering and to save us some CPU cycles.<br/>

`set-prompt` is a function that I use to set the actual prompt. You may not need to call that or whatever equivalent that you have. In my case, I have to do it since my prompt lies on the fancier side of prompts and a change in the amount of characters in the prompts(because an async function is providing its output after the initial prompt display) will need to be taken into account so that's why i run the prompt after everything(all the async functions) has returned, so that the final character lengths are known and correct.<br/>

In the next section, we simply call the init function of the library and then call our own start function after that:

```zsh
async_init
_async_dcpr_start
```

In this section we add our async job to the precmd hook for zsh so that our async job runs on the precmd hook on every prompt. The `precmd` hook is executed before the prompt is displayed.<br/>
Also please do note that this is where we actually tell the async runner what function to actually run. Moreover, this is where we pass any arguments that may or may not be needed by the said function.<br/>
Do keep in mind that the async execution environment our function will run in is not the same as the one your shell prompt will be run in. This means that env vars will not carry over. In our example our docker compose function cannot get the current working directory by just accessing the `$PWD` env var so we will have to pass `$PWD` to it as a function argument manually.<br/>

```zsh
add-zsh-hook precmd (){
  async_job dcpr_info docker_compose_running_pwd $PWD
}
```

This final part serves two purposes. First, it clears the prompt var on changing a directory, so that we don't get a wrong result until we get a new result on a new prompt. Second, this also serves as the definition for our global var that we will use in the prompt.<br/>

```zsh
add-zsh-hook chpwd() {
  dcpr_info_msg=
}
```

Here's everything put together:

```zsh
docker_compose_running_pwd() {
  local cwd="$1"
  local list=$(docker compose ls --format json | jq '.[].ConfigFiles' | tr -d '"')
  local array=("${(@f)$(echo ${list})}")
  for elem in "${array[@]}"; do
    if [[ "${cwd}" == $(dirname "${elem}") ]];then
      echo "\U1F7E6"
      return
    else
        ;
    fi
  done
}

_async_dcpr_start() {
  async_start_worker dcpr_info
  async_register_callback dcpr_info _async_dcpr_info_done
}

_async_dcpr_info_done() {
  #first part
  local job=$1
  local return_code=$2
  local stdout=$3
  local more=$6
  #second part
  if [[  $job == '[async]' ]]; then
    if [[ $return_code -eq 2 ]]; then
      _async_dcpr_start
      return
    fi
  fi
  #third part
  dcpr_info_msg=$stdout
  #fourth part
  [[ $more == 1 ]] || set-prompt && zle reset-prompt
}

async_init
_async_dcpr_start

add-zsh-hook precmd (){
  async_job dcpr_info docker_compose_running_pwd $PWD
}

add-zsh-hook chpwd (){
  dcpr_info_msg=
}
```

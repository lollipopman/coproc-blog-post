# Memoize Commands or Bash Functions with Coprocs!

## Motivation & Some Emoji

How do you speed up a complex Linux iptables generator written in Bash
that executes `ip route` thousands of times, often with the same
arguments?

-   You could rearchitect the program to avoid the duplicate calls?
    Sadly you deem that too difficult.
-   You could return cached results for function calls when the inputs
    are identical, i.e.
    [memoize](https://en.wikipedia.org/wiki/Memoization) the calls? What
    a fabulous idea!

Memoizing `ip route` executions against big routing tables doesn't sound
too fun, so instead let us try to use memoization to speed up a fancy
emoji short code program written in Bash. This program reads standard
input and scans for emoji short codes, based on their text to speech
description provided in Unicode's [Common Locale Data
Repository](https://github.com/unicode-org/cldr-json), thus empowering
you to make your commit messages more beautiful!

``` shell
$ ./emojify <<'EOF'
Remove dead code! :blue_heart: :blue_heart: :blue_heart: :blue_heart: :blue_heart:

This code can go the way of the wonderful :sauropod: as our :nesting_dolls:
tech has been replaced with:

    #!/bin/bash
    msg='Sorry we have gone out of business! :nesting_dolls: :headstone:'
    printf '%s\n' "$msg"
    exit 0

I suppose it is not that easy after all to build a product based on
:nesting_dolls: or to innovate and build a better :mouse_trap:, good luck next
time :blue_heart: :blue_heart:!
EOF

Remove dead code! 💙 💙 💙 💙 💙

This code can go the way of the wonderful 🦕 as our 🪆
tech has been replaced with:

    #!/bin/bash
    msg='Sorry we have gone out of business! 🪆 🪦'
    printf '%s\n' "$msg"
    exit 0

I suppose it is not that easy after all to build a product based on
🪆 or to innovate and build a better 🪤, good luck next
time 💙 💙!
```

The text parser reads each character of input and looks for short codes
of the form `:<text to speech>:` replacing spaces in the text to speech
description with underscores: e.g. `:yo-yo:`(🪀) or `:crystal_ball:`(🔮):

``` bash
function parse-text {
    local cldr_file=$1
    local emoji
    local char
    local parsing_code='false'
    local code_accum=''
    while IFS= read -r -N1 char; do
        if [[ $char != ':' ]]; then
            if [[ $parsing_code == 'false' ]]; then
                printf '%s' "$char"
                continue
            else
                if [[ $char =~ [a-z_-] ]]; then
                    code_accum+=$char
                    continue
                else
                    printf ':%s%s' "$code_accum" "$char"
                    parsing_code='false'
                    continue
                fi
            fi
        else
            if [[ $parsing_code == 'true' ]]; then
                if ! emoji=$(short-code-emoji "$code_accum" "$cldr_file"); then
                    printf 'ERROR: Unable to get emoji :%s:\n' "$code_accum" >&2
                    return 1
                fi
                printf '%s' "$emoji"
                parsing_code='false'
                continue
            else
                parsing_code='true'
                code_accum=''
                continue
            fi
        fi
    done
    return 0
}
```

Once it finds a short code it calls the function `short-code-emoji` to
obtain the short code:

``` bash
if ! emoji=$(short-code-emoji "$code_accum" "$cldr_file"); then
    printf 'ERROR: Unable to obtain emoji for, :%s:\n' "$code_accum" >&2
    return 1
fi
```

The `short-code-emoji` function uses `jq` to query the CLDR json for the
emoji associated with the short code:

``` bash
function short-code-emoji {
    local short_code=$1
    local cldr_file=$2
    local tts=${short_code//_/ }
    local emoji
    if ! emoji=$(
        jq -r --arg tts "${tts}" '
          .annotations.annotations |
            to_entries[] |
            select(.value.tts[0] == $tts) |
            .key' <"$cldr_file"
    ); then
        printf 'ERROR: Unable to parse cldr json\n' >&2
        return 1
    fi
    if [[ -z "$emoji" ]]; then
        printf '?%s?\n' "$short_code"
    else
        printf '%s\n' "$emoji"
    fi
    return 0
}
```

Unfortunately, it is a bit slow, I can watch the emoji paint on my
terminal!

``` shell
$ time ./emojify <input >/dev/null

real    0m0.569s
user    0m0.555s
sys     0m0.017s
```

Parsing a 336Kib json file is pretty quick with `jq` but for every
repeated short code in our input we our repeating our work even though
we know the result. So how can we memoize those repeated function calls
to `short-code-emoji`?

## Bash Functions Are Wacky!

Bash functions are a bit of an odd 🦆 when compared with a traditional
programming language, for example when executing:

``` bash
#!/bin/bash
function epoch {
    printf '%(%s)T\n'
}

unix_epoch=$(date +%s)
printf 'from date cmd: %s\n' "$unix_epoch"
unix_epoch=$(epoch)
printf 'from epoch func: %s\n' "$unix_epoch"
```

You might assume that the calling of `epoch` occurs within the main Bash
process, in contrast to the execution of `date`. However, when a bash
function is executed, within a command substitution, Bash forks a
process and executes the function in a child process or subshell. So
effectively the function is executed as a separate command like `date`
rather than as part of the main Bash process. Try running
`strace --trace=process` on the above snippet to see the forked
children.

Because a function is executed in a separate process, when using command
substitution, a naive memoization strategy for a `rando` function that
returns the same random value for any input might look like this:

``` bash
#!/bin/bash
declare -A CACHE

function rando {
    local IFS=$'\t'
    if ! [[ -v "CACHE[$*]" ]]; then
        CACHE[$*]=$RANDOM
    fi
    printf '%s\n' "${CACHE[$*]}"
}

printf "%s\n" "$(rando butter bubbles)"
printf "%s\n" "$(rando butter bubbles)"
```

But, this will not work as it will output two different random numbers
for the same input because the associative array is only instantiated in
the subprocess and as a consequence any writes to the array are lost
when the subprocess exits. How do we memoize the results if the function
is executed in a separate process?

There are many possible strategies including:

1.  Prime the cache in our main process, to avoid writing to the cache
    in a subshell:

    ``` bash
    #!/bin/bash
    declare -A CACHE

    function rando {
      local IFS=$'\t'
      printf '%s\n' "${CACHE[$*]}"
    }

    OLD_IFS=$IFS
    IFS=$'\t'
    rando_args=("butter" "bubbles")
    if ! [[ -v "CACHE[${rando_args[*]}]" ]]; then
      CACHE[${rando_args[*]}]=$RANDOM
    fi
    IFS=$OLD_IFS
    printf "%s\n" "$(rando butter bubbles)"
    printf "%s\n" "$(rando butter bubbles)"
    ```

    Though that method makes for some pretty hard to read code if you
    need to cache a lot of values!

2.  Cache the results in the filesystem:

    ``` bash
    #!/bin/bash
    CACHE_DIR=$(mktemp -d)

    function rando {
      local IFS=$'\t'
      local key_file="${CACHE_DIR}/$*"
      if ! [[ -e "$key_file" ]]; then
        printf '%s\n' "$RANDOM" >"$key_file"
      fi
      cat "$key_file"
    }

    printf "%s\n" "$(rando butter bubbles)"
    printf "%s\n" "$(rando butter bubbles)"
    ```

    This method works well, but requires cleaning up our files and
    ensuring the underlying block device is fast.

3.  Use Coprocs!

Of course given the title of this blog post we will go with option (3)!

## Grokking Coprocs

A coproc allows us to execute a function as a separate process in a
background subshell and communicate with that process over pipes. If you
squint they are almost like a Goroutine in Go, at least in the message
passing sense. At a more basic level a coproc can be thought of shell
syntactic sugar around named pipes. You daemonize a function as a
separate process and then communicate with that daemon by writing to its
standard input and reading from its standard output:

``` bash
#!/bin/bash

function rando-daemon {
    declare -A cache
    declare -a query
    local IFS=$'\t'
    while read -ra query; do
        if ! [[ -v "cache[${query[*]}]" ]]; then
            cache[${query[*]}]=$RANDOM
        fi
        printf '%s\n' "${cache[${query[*]}]}"
    done
}

rando() {
    local IFS=$'\t'
    local resp
    printf '%s\n' "$*" >&"${RANDO[1]}"
    read -r resp <&"${RANDO[0]}"
    printf '%s\n' "$resp"
}

coproc RANDO { rando-daemon; }

printf "%s\n" "$(rando butter bubbles)"
printf "%s\n" "$(rando butter bubbles)"
```

Here we rename our `rando` function to `rando-daemon`, to indicate it is
now a daemon, and we add a new `rando` function who communicates with
the `rando-daemon` over its stdin and stdout pipes created by the coproc
command `coproc RANDO { rando-daemon; }`. In this invocation `RANDO` is
instantiated as an array of file descriptors connected to the
`rando-daemon` with `${RANDO[0]}` being its stdout and `${RANDO[1]}`
being its stdin. We also retool `rando-daemon` to loop forever reading
from stdin and writing its responses to stdout. With those changes made
the function can now use a local associative array to cache results and
the array persist as long as the coproc is running.

The message format over the pipes is free form, here I chose to use tab
delimited data as it makes for a pretty simple solution.

## Emojify With Coprocs

Given our new knowledge of coprocs we can retool the slow
`short-code-emoji` function as a coproc daemon, adding a while loop
reading from stdin. Our stdin query response is enhanced to include a
tab delimited return code and emoji:

``` bash
function short-code-emoji-daemon {
  local short_code
  local cldr_file
  local tts
  local emoji
  declare -a query
  declare -A emoji_cache
  while IFS=$'\t' read -ra query; do
    cldr_file="${query[1]}"
    short_code="${query[0]}"
    tts=${short_code//_/ }
    if ! [[ -v "emoji_cache[$short_code]" ]]; then
      if ! emoji=$(
        jq -r --arg tts "${tts}" '
          .annotations.annotations |
          to_entries[] |
          select(.value.tts[0] == $tts) |
          .key' <"$cldr_file"
      ); then
        printf 'ERROR: Unable to parse cldr json\n' >&2
        printf '%d\t%s\n' 1 ""
      fi
      if [[ -z "$emoji" ]]; then
        emoji_cache[$short_code]="?${short_code}?"
      else
        emoji_cache[$short_code]=$emoji
      fi
    fi
    printf '%d\t%s\n' 0 "${emoji_cache[$short_code]}"
  done
}
```

Then add a query function to communicate with our daemon:

``` bash
short-code-emoji() {
  local IFS=$'\t'
  local ret
  local resp
  printf '%s\n' "$*" >&"${SHORT_CODE_EMOJI[1]}"
  read -r ret resp <&"${SHORT_CODE_EMOJI[0]}"
  printf '%s\n' "$resp"
  return "$ret"
}
```

Here we improve the `short-code-emoji` query function a bit to allow the
daemon's response to include the return code as well as the emoji. This
allows the query function to receive errors from the coproc daemon and
return them in the same manner as a typical Bash function.

## Benchmarking Emojify

So did our coproc improve the speed of our fancy emojifier?

``` shell
$ time ./emojify <input >/dev/null

real    0m0.239s
user    0m0.015s
sys     0m0.006s
```

Indeed, over a 40% speedup! Now we don't have to wait for our emoji to
render!

``` shell
$ echo ':raising_hands:' | ./emojify
🙌
```

Using this same technique on our iptables generator gained us a speedup
of 2.4 seconds!

## Conclusion

Bash functions are strange and attempting to view them akin to functions
in typical languages will cause a lot of pain in my experience. However,
viewing functions as separate commands exposes the nice symmetry between
commands you execute in Bash and your own functions. Coprocs are an
interesting and fun way to extend that view of Bash functions and turn
them into long running daemons.

## Further Reading

1.  [emojify - full source code both with and without
    coprocs](https://github.com/lollipopman/coproc-blog-post)
2.  [Bash manual on
    coprocs](https://www.gnu.org/software/bash/manual/html_node/Coprocesses.html)
3.  [How do you use the command coproc in various
    shells?](https://unix.stackexchange.com/a/86331/83704): Wonderful
    reply by Stéphane Chazelas on the topic of coproc usage.

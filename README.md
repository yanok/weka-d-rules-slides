This document provides a practical guide to Weka-specific Bazel D rules.

Slides: https://yanok.github.io/weka-d-rules-slides

## Cheatsheet

Let's start with a quick overview of the most common rules you'll be using. This section is a quick reference for the most commonly used rules. It's designed to be a starting point for developers who are new to the Weka-specific Bazel rules.

### `d_library_flavors`

This rule creates four flavors of a D library, corresponding to 4 Weld flavors:

| Suffix            | Meaning                               | Weld suffix |
| ----------------- | ------------------------------------- | ----------- |
| `notrace`         | No instrumentation/tracing, no testing | `Dsit`      |
| `trace`           | Instrumentation/tracing, no testing    | `DsIt`      |
| `notrace_fortest` | No instrumentation/tracing, with testing | `DsiT`      |
| `trace_fortest`   | Instrumentation/tracing, with testing  | `DsIT`      |

The naming convention for the flavors is important. It helps to differentiate between the different builds and their purposes. The Weld flavors are mentioned for context, as they are the inspiration for this system. A fun fact: I initially used Weld-inspired suffixes, but learned the hard way that APFS is case-insensitive!

### `d_library_flavors` Attributes

Here are the attributes for `d_library_flavors`:

-   `name`: The name of the target.
-   `srcs`: The source files for the library.
-   `exports`: Files visible to dependents. If not set, all `srcs` are exported.
-   `exports_no_hdrs`: Files visible to dependents, but excluded from header generation.
-   `dont_hdrgen`: An escape hatch to disable header generation for the library completely.
-   `deps`: Flavored dependencies (other `d_library_flavors`).
-   `non_flavored_deps`: Non-flavored dependencies (`d_library`, `cc_library`).
-   `test_deps`: Additional dependencies for `_fortest` flavors.
-   `tracing_deps`: Additional dependencies for `_trace` flavors.

The `exports_no_hdrs` and `dont_hdrgen` attributes are important for managing header generation, which can be a complex topic. The `non_flavored_deps` attribute exists for historical reasons and will likely be merged with `deps` in the future.

### `d_library_flavors` Example

Here is a simple example of how to use the `d_library_flavors` rule. It shows how to specify sources, dependencies, and test-specific dependencies.

```python
d_library_flavors(
    name = "my_lib",
    srcs = [
        "my_lib.d",
        "another_file.d",
    ],
    deps = [
        ":another_lib",
    ],
    non_flavored_deps = [
        "//third_party:some_c_lib",
    ],
    test_deps = [
        ":lib_used_only_in_uts",
    ],
)
```

### `weka_d_binary`

This rule creates two flavors of a D binary (`notrace`/`trace`) and runs `trace_updater` on the traced one. The `name`, `srcs`, `deps`, and `non_flavored_deps` attributes are the same as in `d_library_flavors`. This rule is used to create executable binaries. The trace_updater is a Weka-specific tool that processes the traced binary.

#### Example

```python
weka_d_binary(
    name = "my_app",
    srcs = ["main.d"],
    deps = [
        ":my_lib",
    ],
)
```

### `weka_d_test`

This rule creates a D test and a wrapper script to run it with a package name as an argument. The `name`, `srcs`, `deps`, and `non_flavored_deps` are the same as in `d_library_flavors`. The `args` attribute allows adding extra arguments to run the test. The wrapper script is necessary to handle the package name argument, which is used to filter tests. The `args` attribute is currently appended to the generated package name argument, but it should probably replace it instead.

#### Example

```python
weka_d_test(
    name = "my_lib_test",
    srcs = ["test.d"],
    deps = [
        ":my_lib",
    ],
    args = ["--verbose"],
)
```

## Weka-specific challenges

Now, let's discuss some of the challenges we face with our build system. This section will cover some of the unique challenges we face at Weka and how we are using Bazel to address them.

### Flavors

We support 2 out of 4 Weld flavors. I believe we don't need to support the other two.
 - Dev vs Release is controlled by "compilation options" or `-copt` flag in bazel
 - static can be enabled per target (requires rules support)

Bazel's explicit dependencies can cause additional pain with flavors.

Consider this code snippet:

```d
version(unittest) {
  import weka.something.something; // Not used anywhere else
}
```

We have a few options to handle this:

-   Union all dependencies, adding `weka-something` as a dep to *all* targets. This is bad for build performance.
-   Specify dependencies for each flavor separately. This is tedious.
-   Do something in the middle (we are here).

We've chosen a middle ground that provides a balance of flexibility and simplicity.

### Instrumentation & Trace-updater

-   **Instrumentation**: This is just code generation, which is pretty easy to support in Bazel.
-   **Trace-updater**: This is just an extra step, running the custom tool, so easy to support as well. Though there are issues with trace-updater being slow and thinking it can write to the global cache.

Together with the flavors, these are the key features of our build system. Bazel's extensibility makes it easy to integrate these custom tools.

### Different build options

We have several build options, including:

-   `tinyweka`
-   `by_exe`
-   `new_traces`
-   ...and more

None of these are currently supported, but it's planned! We will use Bazel's `config_setting` feature for this. This is a forward-looking statement about how we plan to support different build configurations in the future. Bazel's `config_setting` is a powerful feature that will allow us to manage these different build options in a clean and scalable way.

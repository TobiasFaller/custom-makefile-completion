Custom Makefile completion
==========================

This patch introduces the ability to have a custom Makefile target that generates auto-completion input for the bash shell.
With this feature it is possible to customize the completion list that is displayed in the shell.

The Makefile target returns a space and newline separated list of custom completion options.
Since the Makefile has full control over the returned list it can add custom build targets which are not directly specified as a Makefile target.
Because this might have some security implications - executing the Makefile target on auto-completion - the feature is disabled by default.

Configuration options:

- `BASH_COMPLETION_MAKE_ENABLE_MAKEFILE_TARGET`: Enables the feature (Default is "0")
- `BASH_COMPLETION_MAKE_MAKEFILE_TARGET`: Specifies Makefile target to execute (Default is ".BASH-COMPLETION")

How to install
--------------

Run the following command inside your bash-completion installation folder to apply the custom extension.
Make sure to set `BASH_COMPLETION_MAKE_ENABLE_MAKEFILE_TARGET` to `1` in your global `.bashrc` or `.bash_profile` configuration to enable / test the feature.
Be aware that this has security implications because activating the auto-completion in untrusted environments automatically executes the `.BASH-COMPLETION` Makefile target.

```bash
#!/bin/sh

wget https://raw.githubusercontent.com/TobiasFaller/custom-makefile-completion/main/custom-makefile-completion.patch
git apply --check --apply custom-makefile-completion.patch
```

How to use
----------

Below is an example Makefile which implements a static list that will be listed if this bash complection is enabled.
The list of available targets is written to the standard output when the `BASH_COMPLETION_MAKE_MAKEFILE_TARGET` is automatically invoked.
The bash completion parses this list and provides auto-completion to the user.

```Makefile
.PHONY: .BASH-COMPLETION

.BASH-COMPLETION:
	@echo 'build/debug/AppA build/debug/AppB'
	@echo 'build/release/AppA build/release/AppB'

build/debug/%: %Main.c
	echo "Building $(@F) (Debug)"
	gcc -O1 -g -o $(@F) $^

build/release/%: %Main.c
	echo "Building $(@F) (Release)"
	gcc -O3 -o $(@F) $^
```

Below is an example Makefile that implements an auto-generated list that is computed from the available bazel targets:

```Makefile
.PHONY: .BASH-COMPLETION

.BASH-COMPLETION:
	@$(eval $@_BAZEL_TARGETS := $(shell $(BAZEL) query 'kind(rule, //src:*)' 2>/dev/null))
	@$(eval $@_BAZEL_TESTS := $(shell $(BAZEL) query 'kind(rule, //test:*)' 2>/dev/null))
	@echo 'build/debug/* build/release/*'
	@echo 'test/debug/* test/release/*'
	@echo '$(subst //src:,build/debug/,$($@_BAZEL_TARGETS))'
	@echo '$(subst //src:,build/release/,$($@_BAZEL_TARGETS))'
	@echo '$(subst //test:,test/debug/,$($@_BAZEL_TESTS))'
	@echo '$(subst //test:,test/release/,$($@_BAZEL_TESTS))'

build/debug/%:
	echo "Building $(@F) (Debug)"
	bazel build --config=debug //src:$(@F)

build/release/%:
	echo "Building $(@F) (Release)"
	bazel build --config=release //src:$(@F)

test/debug/%:
	echo "Testing $(@F) (Debug)"
	bazel test --config=debug //test:$(@F)

test/release/%:
	echo "Testing $(@F) (Release)"
	bazel test --config=release //test:$(@F)
```

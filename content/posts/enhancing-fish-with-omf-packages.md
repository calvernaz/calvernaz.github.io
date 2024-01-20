---
title: "Fish Shell: Creating and Sharing Packages with Oh My Fish"
date: 2024-01-20T06:50:50Z
draft: true
description: "Dive into the world of Fish shell, a smart and user-friendly command line interface. Learn how to enhance your Fish experience by creating and sharing your own packages using the Oh My Fish (OMF) framework."
tags: ["Fish Shell", "Oh My Fish", "Shell Scripting", "Package Management", "Command Line"]
categories: ["Development Tools", "Productivity"]
---


One of the aspects I personally find most compelling about [fish shell](https://fishshell.com) is its scripting ease. Unlike traditional shells, fish's scripting language is straightforward, clean, and comes with features that encourage writing and maintaining scripts effortlessly. From its intelligent autosuggestions to the robust error handling, [fish shell](https://fishshell.com) stands out as a powerful tool in the developer's toolkit.

With the integration of the [Oh My Fish (OMF) framework](https://github.com/oh-my-fish/oh-my-fish), users can further extend the shell's functionality. [OMF](https://github.com/oh-my-fish/oh-my-fish) offers a package management system, allowing users to install new packages, themes, and utilities, thus customizing their shell environment to their liking.

In this post, we will delve into the process of creating and sharing [fish](https://fishshell.com) packages using [Oh My Fish](https://github.com/oh-my-fish/oh-my-fish), exploring how this powerful combination can elevate your command-line experience to new heights.


## Setting Up Oh My Fish (OMF)

Before we dive into the creation and sharing of packages, it's essential to have Oh-My-Fish (omf) installed in your [fish](https://fishshell.com) environment. `omf` is a package manager for the Fish shell, offering an extensive ecosystem of themes, packages, and a framework for advanced customization. [Follow the installation guide here](https://github.com/oh-my-fish/oh-my-fish#installation).


After installation, you can verify that `omf` is correctly installed by running:
```bash
omf --version
```

Next step, check your existing repositories:
```bash
omf repositories
```
it should output something like `https://github.com/oh-my-fish/packages-main master`

Also, check the packages installed - if it's a fresh installation you won't have much 😊

```bash
omf list
```

Last but not least, check where all the `omf` configuration lives, the environment variable `OMF_CONFIG` will point to
the configured directory.
```bash
env | grep OMF_CONFIG
```

## Creating a Fish Package

In the world of `omf`, what we generally refer to as a package is known as a plugin. These plugins are essentially extensions or add-ons that enhance the functionality of the Fish shell, ranging from simple utility functions to complex integrations with external tools.

### Understanding the OMF Directory Structure

Before we move into the practical steps to create a plugin, it's essential to understand the directory structure that it uses. This knowledge will help you manage and navigate your fish shell environment more effectively. Let's clarify the roles of two key directories: `OMF_CONFIG` and `~/.local/share/omf`.

#### 1. The `OMF_CONFIG` Directory

- **Location and Purpose**: Typically found at `~/.config/omf`, this directory is the heart of your user-specific Oh My Fish configuration. It's where your personalized settings, such as custom themes or packages, reside.

- **User Customization and Flexibility**: This is your playground for customization. Whether you're adding unique plugins or tweaking themes, your changes are encapsulated here, keeping your individual setup neatly organized.

- **Portability and Backup**: Given its nature, the `OMF_CONFIG` directory is perfect for backing up or transferring. Copying this directory allows you to replicate your Oh My Fish setup on another machine seamlessly.

#### 2. The `~/.local/share/omf` Directory

- **Core Installation**: This directory houses the core of your Oh My Fish installation. It includes the package manager and the default sets of packages and themes provided by OMF.

- **Package Storage**: Beyond the package manager, this directory stores the installed packages and themes. Unless overridden by user-specific configurations in `OMF_CONFIG`, the content here forms the base of your Oh My Fish environment.

- **Separation of Concerns**: This directory ensures a clear distinction between the OMF base installation (shared and consistent across users) and user-specific configurations (personalized and varied). Such separation is key for a clean and stable setup.


#### Understanding OMF Plugins

Now that we've familiarized ourselves with the foundational directory structure of OMF, it's time to delve deeper into the anatomy of an OMF plugin.

- **Plugin Directory**: Each plugin resides in its own directory within the `~/.local/share/omf/pkg/` directory.
- **Plugin Files**: A typical plugin may contain several files, but there are a few key ones:
  - `init.fish`: The initialization script that's run when the plugin is loaded.
  - `uninstall.fish`: (Optional) A script that's run when the plugin is removed.
  - `functions/`: A directory containing Fish functions that the plugin provides.
  - `completions/`: A directory containing completion definitions for the commands provided by the plugin.

### Steps to Create an OMF Plugin

   The name of the directory should match your plugin's name. The plugin we will build, as an example, is called `env_loader`, that automatically loads environment variables from the `.env` file when navigating to the directory.

1. Creating the plugin:

```bash
omf new plugin env_loader
```

This should result on the following:

```shell
 create  completions/env_loader.fish
 create  functions/env_loader.fish
 create  init.fish
 create  LICENSE
 create  README.md
 create  uninstall.fish
```

The first thing I do is initialize a git repository from it, but you can also copy this folder and move it to your code directory, because we will create a repository for it and install it from the repository.

```shell
git init .
git commit -am "inital import"
```

2. Write the `init.sh` script

```bash
# reacts to directory changes
function load_env --on-variable PWD
    # Call the load_env function from load_env.fish
    env_loader
end

# Trigger load_env on shell start
emit load_env
```

3. Write the `env_loader` function on the `functions/env_loader.fish` file:

```bash
function env_loader
    set -l env_file "$PWD/.env"
    if test -f $env_file
        for line in (cat $env_file)
            set -gx (string split "=" $line)[1] (string split "=" $line)[2]
        end
    end
end
```

Before distributing your plugin, it's indeed a good practice to thoroughly test it. Sourcing `init.fish` and the functions from `functions/env_loader.fish` will help you ensure that everything is working as expected.

```shell
source init.fish
source functions/env_loader.fish
```

4. Test it!

```shell
mkdir tmp; echo "FOO=BAR" > tmp/.env
```

Navigate into `tmp/` and check the loaded environment variable:
```shell
cd tmp; env | grep FOO
```

Finally, commit the final version and make it available on github, for example `https://github.com/username/env_loader`.

## Sharing Your Package with the Community

Here are two primary ways to share your Fish shell, **submitting to the official Oh My Fish registry** or **creating our own package repository**. We will explore the latter.

1. **Initialize a New Repository**:
   Create a new Git repository on GitHub for example and called it `fish-packages`

2. **Organize Your Packages**:
	 Create a directory inside	`fish-packages` called `packages`. Inside the `packages` directory create a file called `env_loader` with the following content:

```bash
type = plugin
repository = https://github.com/username/env_loader
description = Package provides a way to automatically load environment variables from .env
```

Should end up with file tree like this:
```shell
.
├── README.md
└── packages
    └── env_loader

1 directory, 2 files
```

3. **Inform Your Users**:
Update your `README.md` to include instructions on how users can add your repository as a custom repository to their Oh My Fish installation:
```shell
omf repository add <your_repository_url>
```

List the repositories, notice we have our own repository now:

```shell
https://github.com/oh-my-fish/packages-main master
https://github.com/username/fish-packages master
```

Finally, install the new plugin:
```shell
omf update; omf install env_loader
```

The installation will install the plugin on `~/.local/share/omf/pkg` and will be ready to use!

## Conclusion

Dive in, start creating, and become an active participant in the ever-growing ecosystem of the Fish shell and Oh My Fish. Happy scripting!

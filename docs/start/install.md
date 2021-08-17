# Set up Beaker

## Join

First, sign up for your [Beaker](https://beaker.org) account. If you are an AI2 employee please use 
Google authentication and sign in with your allenai.org email address.

1. From Beaker.org, click **Log In**.
2. From Beaker Public, click **Sign Up**.
3. Choose **SIGN UP WITH GOOGLE** and proceed to create and confirm your Beaker account.

If you already have an account, make sure to sign in before using Beaker.

1. From Beaker.org, click **Log In**.
2. From Beaker Public, click **LOG IN WITH GOOGLE**.
3. Choose the Google account with which you signed up for Beaker.

When signed in, you'll see your user name in the top-right corner of [Beaker.org](https://beaker.org)
and have access to your **Experiments**, **Datasets** and **Groups** pages.

## Install

There are a few different ways to install Beaker:

- On Linux, download the latest [release](https://github.com/allenai/beaker/releases) and extract it.

    ```bash
    curl -s https://api.github.com/repos/allenai/beaker/releases/latest \
      | grep browser_download_url \
      | grep linux \
      | cut -d : -f 2,3 \
      | tr -d \" \
      | wget -qi -
    sudo tar -xvzf beaker_linux.tar.gz -C /usr/local/bin
    rm beaker_linux.tar.gz
    ```

    Place the binary executable in your PATH (e.g., /usr/local/bin/ or ~/bin).

- macOS users can install Beaker through [Homebrew](https://brew.sh/) with a custom tap:

    ```bash
    brew tap allenai/homebrew-beaker https://github.com/allenai/homebrew-beaker.git
    brew install beaker
    ```

- You can also install Beaker directly from source using [Go](https://golang.org/) standard tools.

    ```bash
    go install github.com/allenai/beaker/cmd/beaker@latest
    ```

## Configure and Test

To set up Beaker, log in to [beaker.org](https://beaker.org) and follow the instructions in your
[account](https://beaker.org/user) page.

## Docker

You will need to install Docker to use Beaker. Docker provides a repeatable environment for your
application so Beaker can run it on any machine.

Please install the appropriate Docker Desktop version from the
[Docker site](https://www.docker.com/products/docker-desktop), following Docker's instructions.

## Next step

When you have your Beaker.org account, Beaker, and Docker each installed and configured, proceed to
[your first experiment](run.md) to learn the fundamentals of experiments with Beaker.

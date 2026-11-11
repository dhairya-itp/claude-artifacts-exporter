# Tests

Each folder is a test case in the format `claude plugin eval` reads. The prompts use placeholders such as `<link to one of your Slides decks>`: replace them with links to artifacts you own before running. The tests only read from claude.ai. They export into the working folder and never change your artifacts.

Run from the skill folder (Claude Code v2.1.269 or later; these are real, billed model runs):

    claude plugin eval . --allow-tools Write,Edit,Bash

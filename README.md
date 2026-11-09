# omarchy-dual-boot

Grok skill for a fresh Omarchy install that dual-boots Windows. It sets the Limine menu so Omarchy boots by default after 5 seconds, leaves Windows 11 as the other entry, keeps snapshot UKIs off the ESP, and enrolls Secure Boot with sbctl plus the Microsoft keys.

`SKILL.md` is the procedure the agent follows. `scripts/80-flat-boot-menu` is the Limine hook the agent installs. Everything in this repository is English, including the messages that hook prints.

## Install

Put this directory where Grok loads user skills, so `SKILL.md` is at `~/.grok/skills/omarchy-dual-boot/SKILL.md`:

```bash
mkdir -p ~/.grok/skills
git clone https://github.com/andrzejsliwa/omarchy-dual-boot.git ~/.grok/skills/omarchy-dual-boot
```

A copy or a symlink of this directory to that path works the same way. Grok picks up a new skill when the files land on disk. Open a new session if `/omarchy-dual-boot` is not in the slash menu yet.

Run it on the Omarchy machine you want to configure. The agent edits that machine's boot files.

## Run

In Grok:

```text
/omarchy-dual-boot
```

`/skills` lists it too. Grok also loads the skill when you ask for dual boot, a Limine timeout, or this boot menu on a new Omarchy.

The agent writes boot files with `pkexec` or `sudo`. It enrolls Secure Boot keys only while `sbctl status` reports Setup Mode, and it stops for a firmware change when it does not. Read `SKILL.md` before you let it change the boot menu.

## Tests

```bash
make test
```

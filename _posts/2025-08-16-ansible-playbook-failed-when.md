---
layout: post
title:  "Learnings about Ansible playbook failed_when"
date:   2025-08-16 12:00:00 +0800
permalink: /blog/:title
---

Recently, during work, I encountered a case where an Ansible task using the `ansible.builtin.shell` module was not failing when the command returned a non-zero exit code.

This silent failure caused the playbook to continue executing tasks, and then subsequently fail somewhere down the line.

Here is a simplified example of the task that failed silently:

```
- name: Run a command that may fail
  ansible.builtin.shell: ls /nonexistent_directory
  register: command_result
  failed_when:
    - false 
```
Basically, the gist is that we have a command that
1. returns a non-zero exit code, and in that particular run of the playbook, 
2. the conditions under `failed_when` evaluated to `false`.

Turns out, the task does not fail if a non-zero exit code is returned. If the `failed_when` clause is defined, Ansible will only fail the task if the conditions evaluate to `true`.
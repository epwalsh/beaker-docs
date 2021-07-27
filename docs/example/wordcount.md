# Count words

First, create an account at [beaker.org](https://beaker.org) and follow the instructions in your [account settings](https://beaker.org/user). See [Install](../first/install.md) for more options.

The following example [counts words](https://beaker.org/im/im_qbjvcda1sed7) in the text of [Moby Dick](https://beaker.org/ds/ds_1hz9k6sgxi0a).

```yaml
version: v2-alpha
tasks:
- name: count-words
  image: 
    beaker: examples/wordcount
  datasets:
  - mountPath: /input
    source:
      beaker: examples/moby
  result:
    path: /output
  context:
    cluster: ai2/example
```

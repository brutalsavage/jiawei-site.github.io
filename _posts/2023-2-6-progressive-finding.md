---
layout: post
title:  "Progressive bug finding in the open-source of Deep Learning"
date:   2023-02-06 00:12:40 -0600
tags: bugs
categories: blog
layout: post
---

## Motivation

I build tools to catch bugs in Deep-Learning (DL) toolchains, in order to facilitate the user experience, safety and performance.
To empirically evaluate the usefulness techniques, I report the bugs that my testers found to the open-source (e.g., GitHub Issues) and ask for validation from the developers.
Getting more bugs being fixed makes me feel good: (i) my research is "saving the world"; and (ii) my paper looks stronger.
As my testers find more and more bugs, it is important for me to properly and effectively report these bugs so that I can make sure that I am not wasting time spamming the issue tracker which might eventually made me "blacklisted" by losing my credits in open-source and academia.

I started to pay more attention in reporting bugs after reading John Regehr's article[^regehr] (*"Responsible and Effective Bugfinding"*) a year ago.
While I still strongly recommend everyone to read it, in this article I also want to compile a list of "yeas and nays" from my own, from a more technical perspective, either general or specific to the DL community.

## S1: Compile good reports

### Rules

In addition to some cliche, such as **reproducibility**, **using an issue template** and **pasting the log**, there are more to consider:

1. **Minimization**: Test-case reduction papers[^creduce] tell us that a small bug report is easier for diagnosis. This is very important: making bugs easier to investigate improves the chances of letting developers fix it timely.
2. **Mutation**: Mutate your test-cases and observe if the bug still manifests themselves -- giving developers more improves the chance of instant fixes.
3. **Cleaness**: For example, to optimize Signal-to-Noise ratio, if some logs or environment stuffs are too long, we can use the HTML template below to make it foldable:

```html
<details><summary><i>Click to expand.</i></summary>
<div>

Some very very long text.

</div>
</details>
```

which will look like:

<details><summary><i>Click to expand.</i></summary>
<div>

Some very very long text.

</div>
</details>

**To conclude**, a good bug report should be *informative* by providing rich context, while making it tidy by arranging them according to priority.
This lowers the "bar" for getting contributors to start investigation.
Notably, it is also possible to automate these process, with tools such as using delta debugging (for reduction).

### Example

I taught these to a undergrad collaborator [Jinjun](https://co1in.me/).
Let's look at one *high-priority* bug he previously reported[^colinbugs]:

```python
import torch

p0 = torch.tensor([[4.9334, 5.5571]])
p1 = torch.tensor([[4.5627, 5.6945]])
# torch.rand(1) # no error
# p0 = torch.rand(2) # ERROR

def fn():
    v7 = torch.cat([p0, p0], dim=0) # ERROR
    # v7 = torch.cat([p1, p1], dim=0) # ERROR
    # v7 = torch.cat([p1, p0], dim=0) # ERROR
    v1 = torch.mul(v7, v7) # v1: (5, 2)
    return v7, v1

ret_eager = fn()

compiled = torch.compile(fn)
ret_compiled = compiled()

assert torch.allclose(ret_eager[0], ret_compiled[0]), '\n'.join(map(str, ["", ret_eager[0], ret_compiled[0]]))
# ^^^ no error
assert torch.allclose(ret_eager[1], ret_compiled[1]), '\n'.join(map(str, ["", ret_eager[1], ret_compiled[1]]))
''' ^^^ WRONG!
Traceback (most recent call last):
  File "/home/colin/bug.py", line 23, in <module>
    assert torch.allclose(ret_eager[1], ret_compiled[1]), '\n'.join(map(str, ["", ret_eager[1], ret_compiled[1]]))
AssertionError:
tensor([[24.3384, 30.8814],
        [24.3384, 30.8814]])
tensor([[0., 0.],
        [0., 0.]])
'''
```

- [x] **Minimization**: this test-case is already minimized;
- [x] **Mutation**:
    - L5~6: some shapes are wrong, while others are not;
    - L10~11: test if it is related to alias analysis, turning out to be not；
    - L20~21: test which of the outputs are wrong, turning out to be the second only.
- [x] **Neatness**:
    - L12: annotate the shape of the tensor (so that developers do not need to compute it manually);
    - L23~32: print error messages (also remember to shorten the path!).

From my personal taste, it could be made even better:

```python
import torch
from numpy import testing

i0 = torch.ones((1, 2))  # -> (1, 2)
i1 = torch.ones((1, 2))  # -> (1, 2)
# ❌ Using shape = (1, 2) or (2,)
# ✔️ Using shape = (1,)

def fn(v0, v1):
    v2 = torch.cat([v0, v1], dim=0)  # -> (2, 2) ❌ ERROR
    v3 = torch.mul(v2, v2)           # -> (5, 2)
    return v2, v3

ret_eager = fn(i0, i1)
ret_compiled = torch.compile(fn)(i0, i1)
# ❌ fn(i0, i0) or fn(i1, i1)

testing.assert_allclose(ret_eager[0].numpy(), ret_compiled[0].numpy())
# 👆 ✔️ 1st output is correct. 👇 ❌ 2nd output is wrong!
testing.assert_allclose(ret_eager[1].numpy(), ret_compiled[1].numpy())
# NOTE: Output is undeterministic 🤪
# Mismatched elements: 4 / 4 (100%)
# Max absolute difference: 6.0096755
# Max relative difference: 0.85734004
#  x: array([[1., 1.],
#            [1., 1.]], dtype=float32)
#  y: array([[6.684686, 0.      ],
#            [7.009676, 0.      ]], dtype=float32)
```

This includes (i) using some emojis and squeezing some lines for readability; (ii) using `np.testing.assert_allclose` to render the output differences[^torchclose]; and (iii) use `torch.ones` instead of some hard-coded values to indicate that this bug is not related to the values of the tensors.

## S2: Understanding "customer's" requirement

IMHO, the most important "number" for evaluating bug finding is the **number of bugs fixed**[^sec].
Because (i) it measures the real-world impact / consequence of the technique; and (ii) the metric is very well defined (other ones may have various interpretations).
For example, papers in the hall of fame, such as EMI and YarpGen, have a fixing rate of around 60~70%.
To learn from those award-winning work, we want to make our testers to focus on bugs that are likely to fit developers' interest.
According to my experience in DL communities[^exp], there are a few patterns of reports I found appealing:

1. **Emerging components:** The DL systems are ever emerging. For example, both PT2 and TF3 are targeting better support of compilation and distributed computing. Reports on these topics are likely to get more attention and are more important in terms of real-world impact (reports on them speed up the stablization of such components so they can be used out of fear more timely).
2. **Easy-to-fix bugs:** Some bugs can get immediately fixed when the developer found it easy to fix. For example, some crash bugs from invalid API usages (due to lack of input checking), get fixed quickly as the fixes can just be adding more checkers for prevent such ill-formed conditions.

## S3: Random

**Lastest versions.**
When finding bugs, we should run the fuzzer on the latest versions. Many packages have some channels for distributing nightly packages:

```shell
pip install --pre tf-nightly --upgrade
pip install --pre torch --index-url https://download.pytorch.org/whl/nightly/cpu --upgrade
```

When reporting bugs, we should also check if they are reproducible on the latest versions.
It is not recommended (strongly, IMHO) to *intentially* report out-dated bugs just for letting the developers confirm these bugs so that the number of "confirmed" bugs in the paper looks "better", given that it is a distractive to the communities.

**Responsible.**
More importantly, since the preference of communities can vary, it is important to get feedbacks after some reports, via either communication or "inference" (say have any of your first *N* reports been fixed?).
If such feedbacks are negative, then something is wrong and we should carefully re-evaluate our testing strategy before feeding more undesired reports (Prof. Regehr's article[^regehr] also mentioned this point).
It is also a way to show our respect to the community -- responsible bug reporting "lives longer" in the community.
It is possible that fuzzing bugs are not prioritized for not being user-facing or just the lack of "manpower" in the community.

**Patch it if you can.**
Oftentimes developers in DL communities are very busy and may not have time to fix non-user-facing bugs.
In such cases, reporters are often asked "any PRs are welcome" (or "patches are welcome").
Sometimes, such bugs might not be that hard to fix: according to my experience in TVM, 
some bugs, say the integer mismatches, can be easily fixed in a few minutes as they just need a casting[^cast].
This point is also suggested in Regehr's article[^regehr].


## Disclaimers

(i) The list of suggestions are not targetting security bugs; and (ii) "Views are my own" is applied to all my posts, including this one (So don't assume I am speaking for anyone, lol).


[^regehr]: [Responsible and Effective Bugfinding](https://blog.regehr.org/archives/2037)
[^creduce]: [Test-Case Reduction for C Compiler Bugs (Most Influential PLDI Paper for 2012!)](https://users.cs.utah.edu/~regehr/papers/pldi12-preprint.pdf)
[^colinbugs]: [[pt2] compiled function with cat and mul gives wrong results](https://github.com/pytorch/pytorch/issues/93365)
[^torchclose]: [`torch.testing.assert_close`](https://pytorch.org/docs/stable/testing.html#torch.testing.assert_close) is also a chioce but I prefer numpy's for a better error message.
[^sec]: Disclaimer: I am not a security researcher, it might not be applicable to security bugs.
[^exp]: Based on my experience (200+ bugs reported) in recent year on PyTorch, TensorFlow, TensorRT, TVM, etc.
[^cast]: [[Fix] int32/64 mismatch of buffer elem_offset at HandleBufferBindScope](https://github.com/apache/tvm/pull/11755/files)

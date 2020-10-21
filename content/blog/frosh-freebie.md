+++
title = "How Many First-Years Are Taking Their Free Course?"
date = 2020-10-20
tags = ["Brown University"]
draft = false
+++

To de-densify campus, Brown University [moved to a three-term model](https://www.brown.edu/news/2020-07-07/healthy) in which students take courses for *two* of three possible terms. This third term will be carved out of the summer months of 2021, during which an unlucky subset of the student body will swiftly discover that Brown does *not* air-condition its dorms. So, who are these unenviable souls? Well, first-years, *obviously*, who will begin in the spring semester and continue through the summer.

To compensate them for this inconvenience, Brown has extended all first-years the opportunity to remotely take one course this Fall, for free! **How many took Brown up on this offer?**

<!-- more -->

To compute this, we'll mine data from [Courses@Brown](https://cab.brown.edu/) (Brown's course-enrollment portal) focusing on the `regdemog_json` field:
```json
{
  ...
  "regdemog_json": "{\"So\":13,\"Jr\":5,\"Oth\":5,\"F2\":93}",
  ...
}
```
This field tells us that the enrollment of [CSCI0190](http://cs.brown.edu/courses/csci0190/2020/) consists of 13 sophomores, 5 juniors, 5 non-undergraduate students, and 93 incoming first-year students.

Since each first-year is permitted to take *at-most* one course, the total number of first-years taking this opportunity is equal to the sum of first-year enrollments across all courses.

As with my [last blog post](../class-size-paradox) analyzing C@B, we compute this with the help of [`jq`](https://stedolan.github.io/jq/):

```bash
$ jq -s '
    map(if (.regdemog_json | length) > 0
        then
            (.regdemog_json | fromjson | .F2)
        else
            0
        end) | add
    ' db/202010/*.json

2021
```

**2,021 first-years are taking a course remotely this semester!**

How does this compare with the *total* number of incoming first-year students? Well, uh, *I'm not sure*. I don't think Brown has released its fall headcounts yet. ~~I'll update this post once they do.~~ (**Update 2020-10-21:**) The total number of first-year students is in the ballpark of 1,600.

However, according to a [March press-release](https://www.brown.edu/news/2020-03-26/admitted) Brown anticipated that only 1,665 prospective students would accept their admissions offers (out of 2,533). **Did Brown grossly underestimate its yield rate?** (**Update 2020-10-21:**) No, given the total number of first-years, this is almost certainly an issue with double-counting students.

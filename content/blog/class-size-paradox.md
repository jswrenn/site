+++
title = "Brown’s Class Size Paradox"
date = 2020-10-19
tags = ["Brown University"]
+++

University Admissions departments love to brag about their schools’ small class sizes. Brown University, for instance, [boasts](https://admission.brown.edu/explore/admission-facts) a 7:1 student–faculty ratio (so “you’ll have a lot of face-to-face time with some of the best researchers and teachers in academia”) and that “72 percent of our undergraduate classes have fewer than 20 students.” These claims invite prospective students to daydream of cozy, intimate classrooms, and personalized learning experiences — dreams quashed (or perhaps squished?) on one’s first day of classes.

Nobody is lying, *per se* — this is merely an instance of the “class size paradox”: the size of the average class is relatively small, but the experience of the average student suggests otherwise.

<!-- more -->

To see this in action, we’ll mine data from [Courses@Brown](https://cab.brown.edu/), Brown’s course-enrollment portal. The portal’s internal API reports up to three potential indicators of class size:
```json
{
  /* other fields */

  /* 30 - 14 = 16 seats taken  */
  "seats": "<strong>Maximum Enrollment:</strong> <span class=\"seats_max\">30</span> / <strong>Seats Avail:</strong> <span class=\"seats_avail\">14</span>",
  /* 10 + 5 + 1 = 16 seats taken */
  "regdemog_json": "{\"FY\":10,\"So\":5,\"Oth\":1}",
  /* 16 seats taken */
  "regdemog_html": "<p class=\"enroll_demog\">Current enrollment: 16</p><div class=\"demogchart\"></div><div class=\"demogchartlegend\"><ul><li class=\"pieitem item_fy\">62.50% - First Year</li><li class=\"pieitem item_so\">31.25% - Sophomore</li><li class=\"pieitem item_oth\">6.25% - Other</li></ul></div>"

  /* other fields */
}
```
Since some of these indicators may be missing or disagree for a small subset of courses, we’ll simply take the maximum of these three indicators as our authoritative class size count.

## The Average Class

The average class size is merely the total enrollment (across all courses), divided by the number of offered courses. Applied to our dataset:

|    Semester | Size of Average Class |
| ----------: | --------------------: |
|   Fall 2016 |                 23.98 |
| Spring 2017 |                 22.75 |
|   Fall 2017 |                 25.52 |
| Spring 2018 |                 23.31 |
|   Fall 2018 |                 23.81 |
| Spring 2019 |                 22.95 |
|   Fall 2019 |                 24.50 |
| Spring 2020 |                 23.41 |

The average class at Brown is, indeed, small! However, “average class size” reflects the number of students the average *professor* can expect to teach per course — *not* the average number of peers a student should expect to have. Imagine a hypothetical university offering only two courses: one with an enrollment of 100, and another with an enrollment of 0. The average class size is 50, but this is nonsense: 100% of students are in a class of 100!


## The Average Student

If you’re an average student and you pick a course with respect to the distribution of course popularity, how many people (including yourself) can expect to be in that course? To compute this, we square the enrollment of each course in our dataset, reflecting the fact that in a class of size *N*, *N* students will experience a class of that size. Applying this to our dataset we find:

|    Semester | Expected Class Size |
| ----------: | ------------------: |
|   Fall 2016 |               75.43 |
| Spring 2017 |               73.53 |
|   Fall 2017 |               80.11 |
| Spring 2018 |               73.25 |
|   Fall 2018 |               76.90 |
| Spring 2019 |               72.94 |
|   Fall 2019 |               84.02 |
| Spring 2020 |               81.08 |

So, while the size of the average class is relatively small (≈24 students), this “average class” is hardly representative of the experience of the average student.

In light of this, Brown’s claim that “72 percent of our undergraduate classes have fewer than 20 students” fares poorly. Brown’s ample supply of small classes services only a small minority (≈25%) of all enrollments in undergraduate courses:

|    Semester | % of Enrollments in Small (_n<20_) Classes |
| ----------: | -----------------------------------------: |
|   Fall 2016 |                                      25.58 |
| Spring 2017 |                                      27.53 |
|   Fall 2017 |                                      23.85 |
| Spring 2018 |                                      26.49 |
|   Fall 2018 |                                      26.05 |
| Spring 2019 |                                      27.20 |
|   Fall 2019 |                                      25.03 |
| Spring 2020 |                                      22.81 |

Among undergraduate computer science courses — where demand has long outpaced the supply of educators — fewer than *four percent* of enrollments are into classes with less than 20 students!

## Other Shenanigans

This class size paradox is not the only discrepancy between Admissions and reality. The claim that “72 percent of our undergraduate classes have fewer than 20 students” is misleading, even at face value. Such small classes are not quite as prevalent as claimed:

| Semester    | % (_size_<20) |
| ----------: | ------------: |
|   Fall 2016 |         62.09 |
| Spring 2017 |         65.11 |
|   Fall 2017 |         59.98 |
| Spring 2018 |         64.04 |
|   Fall 2018 |         63.11 |
| Spring 2019 |         65.90 |
|   Fall 2019 |         63.20 |
| Spring 2020 |         62.13 |

Brown misses its advertised mark of 72% by a substantial margin in all years for which data is available. I suspect they reach that figure by including graduate and independent study courses.

Similarly, the claim that Brown's student-faculty ratio is 7:1 (and that "100 percent of our faculty teach undergraduates", as Admissions asserts in the same sentence) gives the misleading impression that each educator needs only mind seven students. It is flatly incorrect that 100% of faculty members teach undergrads: One does not need to strain hard to find adjunct faculty with no teaching load, and Brown’s clinical faculty are entirely excused from undergraduate teaching. Among the faculty who do teach, not all teach every semester: professors may take leaves, sabbaticals, and “buy out” some of their required course load.

## Technical Addendum

The Courses@Brown API produces JSON, whose fields are often either HTML or JSON-as-a-string. Whenever I am faced with this sort of hard-to-parse mixed-representation data, I reach for two composable utilities: [`jq`](https://stedolan.github.io/jq/) and [`xidel`](http://www.videlibri.de/xidel.html). For instance, to produce the enrollment count from `.seats`, I pipe the API's JSON for a course into this bash function:
```bash
function enrollment_from_seats_html() {
  jq -rc '"<parent>" + .seats + "</parent>"' \
    | xidel - -s --xpath='
        [
          //span[@class="seats_max"]/text(),
          //span[@class="seats_avail"]/text()
        ]' \
    | jq -c '
        if (. | length == 2) then
          (.[0] | tonumber) - (.[1] | tonumber)
        else
          null
        end'
}
```
Perhaps I’ll go into in more depth about the technical approach behind this blog post in a future entry!

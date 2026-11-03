# Reflections — TD2 Smart Mobility Le Mans

## 1. Expected easy vs hard sub-questions
I expected "where is transport supply concentrated?" to be easy, because the
static GTFS schedule answers it directly. "Where is supply too low compared to
demand?" seemed harder, because there is no open ridership data. "When do
recurring delays happen?" and any prediction seemed impossible on day one,
because GTFS-RT has no history: it only becomes answerable after weeks of our
own snapshots.

## 2. Why the static GTFS feed does not refresh every 15 minutes
The schedule only changes when SETRAM publishes a new feed (valid for a few
months). Downloading a ~2.4 MB zip and reprocessing 214,507 stop-times every 15
minutes would produce the same result 96 times a day: wasted compute, and
useless "nothing changed" runs. Monthly (plus manual runs) matches how often the
source actually changes.

## 3. What breaks when ID columns are auto-typed independently
If one file's stop_id is read as int64 ("1001" -> 1001) and another as text
(because it contains "FLEX-381"), the merge compares 1001 with "1001": they
never match. Depending on the pandas version, the join either raises a type
error or, worse, silently returns empty or missing rows, so stops disappear
from the analysis without any warning. Leading zeros ("0123" -> 123) are also
lost. Reading every GTFS file with dtype=str keeps IDs as identifiers, not numbers.

## 4. Per-stop vs per-route logging over a semester
Per stop-time update: ~5,000 rows x 96 snapshots/day x 84 days = ~40 million
rows. Per route: ~27 rows x 96 x 84 = ~220,000 rows, about 180 times smaller and
easy to keep in Git. What becomes harder: knowing *where along a route* delays
appear (which stops, intersections or segments), so a planner cannot target a
specific bottleneck, only a whole line.

## 5. Alternative to the early-good / late-bad colour encoding
I would colour by absolute deviation from schedule, with a diverging scale:
green within +/-60 s, orange beyond 1-3 min, red beyond 3 min, and early
running shown in blue/purple instead of green. In my snapshot, routes 34
(-45.2 s on average), 28 (-35.9 s) and T2 (-27.4 s) currently look like the
"best" lines. Under my rule they would be flagged: a bus leaving early strands
waiting passengers. The planner's decision would change from "ignore these
lines" to "adjust their schedules or add timing points".

## 6. Both halves of the scheduling mismatch
Static every 15 minutes is wasteful: the source changes every few months, so
96 runs a day recompute identical data and add noise to the history. Real-time
monthly is useless: delays change minute by minute and alerts expire within
hours. One snapshot a month is a random moment that cannot show rush-hour
effects, disruptions or recurring patterns. And since GTFS-RT has no archive,
every snapshot we do not take is lost forever.

## 7. Sanity check of one delayed route
In my snapshot of Tuesday 3 November 2026 at 08:40 (Paris time, morning rush
hour), route C4 was the most delayed: 189.6 s average over 20 trips, with one
stop-time update at 2,616 s (about 44 minutes late). No service alert was
active. That single trip explains most of the average: about 30 updates near
2,600 s out of 609 already add more than 120 s to the mean. So it looks like
one real incident on one vehicle (breakdown, traffic) or a stale prediction
for a vehicle that had not started, rather than the whole line running late.
A second snapshot 4 minutes later confirmed it: C4's max delay was still
exactly 2,616 s while its average rose to 215.6 s, so the same single trip
stayed ~44 min late across both snapshots. The network average (15.3 s) is
plausible for rush hour. Lesson: the mean is very sensitive to outliers; the
median and a "% of trips > 5 min late" KPI would be more honest.

## 8. When population misleads as a demand proxy
A commune with few residents can have very high demand if it hosts a large
employer, a shopping or business zone, a hospital, a university campus or an
event venue: thousands of people travel there daily but do not live there.
The reverse also happens: a residential commune with many car-owning
households or elderly residents can have a large population but low transit
demand.

## 9. My supply metric, and a commune where it misleads
Metric: scheduled stop-visits per weekday per 1,000 residents (sum of
daily_trip_count of all stops assigned to the commune, divided by population).
It ranks La-Chapelle-Saint-Aubin first (1,010.3: 53 stops and 2,343 visits for
2,319 residents) and Le Mans only 10th (153.8), which is clearly wrong: Le Mans
is the centre of the network and contains the 5 busiest stops (Gares 250 trips,
Mairie Le Mans, Gare routière, St-Exupéry). The cause is my assignment method:
each stop is assigned to the nearest commune *centre*. Le Mans is large, so
stops in its outer neighbourhoods are closer to a small neighbouring commune's
centre and get counted there (also visible for Coulaines, 761.9, and Rouillon,
714.9). A fix would be point-in-polygon assignment using official commune
boundaries (IGN Admin Express).

## 10. Aggregating stop-level delay events without blowing up the history
Instead of appending every snapshot, I would keep one rolling aggregate file
keyed by (stop_id, weekday, hour), updated in place at each snapshot: count of
observations, sum of delays, sum of squared delays (for the standard
deviation), number of updates over 5 minutes late, and the maximum. Its size is
bounded (about 1,800 stops x 7 days x 24 hours, ~300,000 rows at most, and
much less in practice) whatever the duration of collection, yet it still
answers "which stops are repeatedly late, and when?". Raw snapshots, if
needed, would go to compressed storage outside Git.

## 11. Reflection 1 predictions vs what I actually found
Answerable today: where scheduled supply is concentrated (the hub ranking is
credible), the current delay by route, and active alerts. Needed judgment calls
from me: population as a demand proxy, the nearest-centre commune assignment
(which produced the misleading ranking), the mean as delay KPI, and the colour
encoding. Needs weeks of history: recurring delay patterns by hour or day,
chronically late routes, and any prediction. My initial guess was right about
the history problem, but I underestimated how much the "easy" supply question
depends on a methodological choice (the commune assignment).

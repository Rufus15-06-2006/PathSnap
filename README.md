# PathSnap – Task C: Road graph, HMM and Viterbi

This folder implements Person C's decoder module. It deliberately does **not**
train a classifier: Person B supplies per-observation candidate arc probabilities.

## Dataset format verified from the supplied PathSnap materials

For each trip:
- `.nodes`: two whitespace-separated columns: longitude, latitude.
- `.arcs`: two integer columns: `from_node_id`, `to_node_id`. The zero-based
  row index is the `arc_id`.
- `.track`: longitude, latitude, timestamp in seconds.
- `.route`: reference chronological sequence of arc IDs (development/evaluation
  only; never pass this file to inference).

The supplied archive contains 80 `train` trips and 10 `student_test` trips.
The evaluator partition is not present in the uploaded student archive.

## Decoder contract with Person B

Person B should provide one candidate list per GPS observation:

```python
candidates = [
    [(arc_id_1, probability_1), (arc_id_2, probability_2)],
    [(arc_id_3, probability_3), (arc_id_4, probability_4)],
]
```

Each probability is the classifier's score/probability for that candidate
road arc. Candidate IDs must retain their original `.arcs` row-index mapping.
Probabilities are normalised per observation inside the decoder.

## Minimal usage

```python
from task_c.decoder import RoadGraph, classifier_only, viterbi_decode, route_continuity

graph = RoadGraph.from_files("trip.nodes", "trip.arcs")
# candidates must come from Person B's model output:
top1 = classifier_only(candidates)
decoded = viterbi_decode(
    graph,
    candidates,
    timestamps=gps_timestamps,
    max_transition_hops=4,
    max_gap_seconds=10.0,
)
print(decoded["predicted_arc_ids"])
print(route_continuity(graph, top1))
print(route_continuity(graph, decoded["predicted_arc_ids"]))
```

## Current decoding design

- Hidden state: candidate `arc_id` at each GPS observation.
- Emission: log of Person B's candidate probability.
- Transition: directed road-network reachability between candidate arcs.
  Directly consecutive arcs receive no extra-hop penalty; a short path up to
  `max_transition_hops` is allowed and penalised by path length.
- Disconnected candidates receive a finite penalty, so a sequence can still
  be decoded when the candidate set contains no connected alternative. Such
  transitions are counted in diagnostics and should be analysed.
- Empty candidate rows and non-increasing / large timestamp gaps split the
  sequence into independent chunks.
- Viterbi is computed in log space for numerical stability.

This is a baseline implementation. The transition model and gap threshold
must be tuned using the permitted training and student-testing partitions,
then compared against classifier-only predictions. Do not claim improvement
until measured.

## Route continuity metric

`route_continuity()` reports:
- direct continuity: consecutive predicted arcs share an endpoint node;
- reachable continuity: the second arc can be reached from the first in at
  most `max_hops` directed transitions.

Report both definitions clearly. A high reachable score is not the same as
direct continuity.

## Run tests

```bash
python -m unittest discover -s tests -v
```

## Important integration note

Person D should call `viterbi_decode()` after loading Person B's classifier and
constructing the candidate lists. The decoder returns one predicted arc ID
per input observation (or `None` when no candidate exists). Reference `.route`
labels must stay outside the inference path.

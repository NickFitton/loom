Label: wayfinder:prototype
Type: prototype
Status: resolved
Assignee: Codex

# Prototype the Control Station and decision batches

## Question

What information architecture and interaction model let a non-conversational Control Station manage many active Projects while supporting project drill-down and actionable work? Make a rough but concrete prototype of the portfolio view, run/step visibility, approval points, and decision-batch cards with card-scoped challenge/clarification discussion, then use it to resolve the UX direction with the Owner.

## Prototype asset

Captured on branch `prototype-control-station-and-decision-batches` at `43e577f2678e96319e12804e58c059eb3eb8cb0b`.

## Answer

Use the **Project tree** Control Station direction. The primary workspace is a compact connected hierarchy: each Project owns multiple Issue nodes, each Issue independently shows whether it is running, and an Issue has zero or more actionable child rows. A Project summarizes its count of running Issues. Actions are colour- and icon-coded by work type and the whole row is the control; prototype iteration review and iteration selection occur in native modal workflows with in-memory, line-scoped comments. The Owner selected this direction after reviewing and refining the prototype.

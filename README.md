# 7 Fold Custom Venn Diagram Visualisation Tool
This tool aims to help create a way to visualize multiple intersections of data sets in a Venn diagram using a custom open-source Python library.

## Context & Problem Statement

A company runs 7 parallel marketing campaigns simultaneously. A single customer may participate in any combination — from just one campaign to all seven. Marketing teams critically need to understand:

- How many people are in **exactly one** specific campaign?
- Which **pairs of campaigns** yield the largest overlap?
- Are there customers covered by **all seven** campaigns?

The classical solution — Euler/Venn diagrams with circles — works reliably only for 2–3 sets. With 4 circles, the diagram becomes unreadable: many intersections visually disappear or become impossible to distinguish.

**The challenge:** Design a geometric model that correctly displays all 127 possible combinations for 7 sets, then overlay real customer data onto it.

---

## Project Architecture

The project is logically split into two independent parts.

### Part 1: A Library for Constructing Smooth Curves

A reusable module that creates complex closed contours from Bézier curves.

#### What Was Built

1. **Cubic Bézier Curve** — the fundamental building block. Each curve is defined by four points: start, end, and two control points that "pull" the curve toward them.

2. **An Alternative Curve Definition** — via endpoint slope angles and a "tension" parameter. This implements the Hobby algorithm, borrowed from the METAFONT system. Unlike classic Bézier curves, you don't need to manually tune control points — just specify the entry angle at the start and the exit angle at the end.

3. **Closed Spline** — given a set of points through which a smooth closed curve must pass, the library automatically computes optimal entry/exit angles at each point by solving a system of linear equations. The user only provides point coordinates and, optionally, tension coefficients for each segment.

4. **A Simplified SVG Path Parser** — allows prototyping the shape in a vector editor, exporting to SVG, and having the library convert it into a programmatic object.

5. **Geometric Utilities** — curve rotation, translation, scaling, finding the farthest point from a given location, and removing tiny segments.

**Outcome of Part 1:** A tool that constructs perfectly smooth closed contours from points and angles.

---

### Part 2: Building a 7-Set Venn Diagram

The main script that consumes the library from Part 1 and overlays real customer data.

#### Step 1: Designing the Petal Shape

Core idea: instead of 7 circles, use **one complex shape** repeated 7 times with rotation around a common centre.

The shape is designed as follows:

- An imaginary cylinder is taken. The petal is "cut" and unrolled onto its surface.
- Control points are placed on this unwrapped surface using "row" and "column" coordinates. Row controls distance from the centre; column controls rotation angle.
- At each control point, the curve slope angle is computed as the sum of two components: the tangential angle (tangent to the circle) and a correction for the change in row.
- A spline is constructed from the resulting points and angles using the library from Part 1.
- The finished shape is normalised — scaled and rotated so that all 7 petals fit into a common circle and remain symmetric.

**Key insight:** The shape is not circular but cleverly curved. This allows all 7 petals to intersect each other in every possible combination — something circles cannot achieve.

#### Step 2: Generating All 7 Petals

The base petal is successively rotated by multiples of 1/7 of a full turn. This yields 7 petals, each assigned a sequential number corresponding to one marketing campaign.

#### Step 3: Computing All Intersection Regions

For every bitmask from 1 to 127:

- Determine which petals should be included (bit = 1) and which excluded (bit = 0).
- Take the geometric intersection of all included petals.
- Sequentially subtract all excluded petals from the resulting region.
- If the result is non-empty, store it together with the corresponding mask.

**Mathematically,** this is pure Boolean algebra on sets, implemented through shapely operations.

#### Step 4: Preparing Customer Data

Customer data is loaded from PostgreSQL into pandas. Each row represents a customer, with seven columns as campaign participation indicators (1 or 0).

**Conversion to bitmasks:**  
For each customer, a single number is computed where each bit corresponds to participation in a specific campaign. For example, participation in campaigns #0, #3, and #4 yields the mask `2⁰ + 2³ + 2⁴ = 1 + 8 + 16 = 25`.

Customers are then grouped by mask, and the count of customers per mask is calculated. The output is a dictionary where keys are masks and values are customer counts.

#### Step 5: Visualisation

For each non-empty geometric region:

- Look up the customer count from the dictionary using the corresponding mask.
- Convert the mask into a human-readable string (e.g., `"C0+C3+C4"`).
- Find a representative point inside the region (centroid).
- Dynamically compute font size — smaller regions get smaller text to prevent overflow.
- For certain combinations where automatic positioning causes overlaps, apply manual offsets (e.g., "shift the text for combination C0+C3 upward by 0.8").
- Render the text in two lines: combination name and customer count.


#### Result:
![Result](https://github.com/polinashishkina/7-Fold-Custom-Venn-Diagram-Visualisation-Tool/blob/main/7%20Fold%20Venn%20Diagram/venn_diagram.png)

#### Step 6: Export

The finished diagram is exported as PNG at a specified resolution. An SVG version is also available for embedding in reports.

---

## Technical Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Circles fail to produce all 7-set intersections | Designed a custom petal shape using cylinder unwrapping |
| Smooth curves are hard to tune manually | Implemented the Hobby algorithm — specify angles and tension only |
| Geometric operations on 127 regions can be slow | Used shapely with the optimised GEOS backend |
| Text overlaps region boundaries | Dynamic font sizing + manual offsets for tricky cases |
| Data from different sources | Universal input format: 7 binary columns → bitmask |

---

## Business Value

- See the entire intersection landscape on a single screen
- Get answers in seconds instead of hours of SQL queries
- Quickly test hypotheses: "what if we drop campaign #3?" — just regenerate the diagram
- The curve library is decoupled and reusable for other projects (graphics, animation, CAD)
- Documented and handover-ready code

---

## How Another Developer Can Use This

1. Take the `bezier` library to construct any smooth closed contour.
2. Take the `VennDiagram` class and adapt it to your own 7 sets by replacing the permutation matrix.
3. Feed in any data with 7 binary attributes and get the diagram.

---

## Summary

This project solves a problem for which no off‑the‑shelf solution exists. It combines:

- Applied mathematics (Bézier curves, splines, Boolean geometry)
- Engineering work (database integration, optimisation)
- Product thinking (end‑user convenience)

---

## Tech Stack

Python, shapely, matplotlib, pandas, numpy.

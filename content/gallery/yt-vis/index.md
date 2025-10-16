+++
image = "grids0.png"
date = "2019-01-23"
title = "Volumetric rendering with yt"
type = "gallery"
+++

This page shows volume renderings produced with [yt](https://yt-project.org), a python package for analyzing
multi-resolution volumetric (structured and unstructured) and particle data. Initially written for working with
astrophysical simulation data, yt is now widely used across many disciplines dealing with 3D simulation or
observational/experimental data. yt can be used for data analysis and manipulation, including creating isosurfaces and
streamlines, exporting 3D scenes to interactive viewers such as ParaView and MeshLab, and subsetting data in many
different ways.

This toy animation shows a rotating cosmological volume with grid annotations.

{{< yt 4-oSldNvBhI 72 >}}
&nbsp;

This animation shows a deeply nested zoom in a sample AMR (adaptive mesh refinement) dataset using a custom colourmap,
with 1795 logarithmic steps changing the window from 97.8 kpc (diameter of a large spiral galaxy) down to 9.78e-11 kpc
= 0.0202 AU = 3.02e6 km ≈ 2 R⊙ and encoded at 60 fps.

{{< yt bGqhjTh48_o 72 >}}
&nbsp;

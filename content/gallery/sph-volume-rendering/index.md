+++
image = "b11.png"
date = "2014-12-16"
title = "Volume rendering of SPH particle density"
type = "gallery"
+++

<!-- https://www.computecanada.ca/visualization/sph-visualization -->

The proof-of-concept animation below shows volume rendering of an SPH (smooth particle hydrodynamics) simulation of
galaxy formation by Fabrice Durier (UVic) contaning 12 million particles, with dataset coloured by the density. The goal
here was to render the hydrodynamical variables in a smooth way, despite the particle nature of the dataset. For this
visualization, we constructed a Delaunay tessellation of the particle distribution using <a
href="http://www.qhull.org/">qhull</a> and stored it as an Unstructured Grid VTK file, with 80 million tetrahedral
cells. This file was then read by ParaView and rendered as a volume. The entire movie containing 1800 frames was
rendered on a laptop, taking about 1 minute per frame, but can be easily rendered on a cluster to speed up the process.

{{< yt qyGj66syLnQ 72 >}}
&nbsp;

Note that performing the Delaunay tessellation on particles is useful primarily for volumetric rendering, as other views (slice, clip) are likely to show artifacts from the Delaunay tessellation in the form of long edges. Also, it is important to point out that the Delaunay tessellation is unique for a randomized set of points. For equidistant points sitting on a 3D Cartesian mesh one can pick a Delaunay tessellation, but ParaView will have trouble processing it leading to crashes and/or segmentation errors, so for this type of rendering one must avoid equally spaced particles.

## Workflow details

While qhull has a C++ interface, it is not well documented yet, so we perform this visualization using qhull's command-line tools. Please install qhull on your machine before proceeding.

We assume that particles are stored in a text file called `qhullParticles.txt` in the following format:

```
1801644
0.114304 0.021919 0.021797637
0.112415 0.017302 0.036328580
0.111180 0.015041 0.049563099
...
```

where the first line stores the number of dimensions, the second line the number of particles, and the rest store
particle x,y,z coordinates with one particle per line. The 3D scalar fields (pressure, density, etc.) should be stored
in a file named `qhullFields.txt` with one particle per line as follows:

```
1801644
20698.763671875 1027.830322266
20515.035156250 1013.588073730
...
```

Compute the Delaunay tessellation with the bash command

```
qvoronoi TI qhullParticles.txt TO qhullTetrahedra.txt TF100000 i
```

This will place the faces of the Delaunay tetrahedra into `qhullTetrahedra.txt`. Next compile and run the code to
convert the data to VTK:

```
// File:        delaunayVTK.cpp
// Description: Converts ASCII output of QHull's Delaunay tessellation to VTK Unstructured Grid
// Author:      Alexei Razoumov
// Date:        2014-Dec
//-----------------------------------------------------------------------------
#include <stdio.h>
#include <iostream>
#include <sstream>
#include <fstream>
#include <vector>
#include <string>
#include <vtkHexahedron.h>
#include <vtkFloatArray.h>
#include <vtkPoints.h>
#include <vtkVersion.h>
#include <vtkSmartPointer.h>
#include <vtkTetra.h>
#include <vtkDoubleArray.h>
#include <vtkPolyhedron.h>
#include <vtkCellArray.h>
#include <vtkDataSetMapper.h>
#include <vtkActor.h>
#include <vtkXMLUnstructuredGridWriter.h>
#include <vtkUnstructuredGrid.h>
#include <vtkPointData.h>
#include <vtkVertexGlyphFilter.h>

using namespace std;

int main() {
  float xyz[3];
  FILE *file;
  int i, j, ip, pid[8], nvertices, ncells, nlocal, **cell = NULL, vertex, linenum, itemnum;
  double *x = NULL, *y = NULL, *z = NULL, *p = NULL, *rho = NULL;
  string item, line;
  ifstream inFile;

  cout << "reading tetrahedra ..." << endl;
  inFile.open("qhullTetrahedra.txt");
  linenum = 0;
  while (getline(inFile, line)) {
    linenum++;
    if (linenum == 1) {
      istringstream linestream(line);
      while (getline(linestream, item, ' '))
	ncells = atoi(item.c_str());
      cell = new int*[ncells];
    }
    if (linenum > 1) {
      istringstream linestream(line);
      ip = linenum - 2;
      cell[ip] = new int[4];
      itemnum = 0;
      while (getline(linestream, item, ' ')) {
	cell[ip][itemnum] = atoi(item.c_str());
 	itemnum++;
      }
    }
  }
  inFile.close();

  cout << "reading particles ..." << endl;
  inFile.open("qhullParticles.txt");
  linenum = 0;
  while (getline(inFile, line)) {
    linenum++;
    if (linenum == 2) {
      istringstream linestream(line);
      while (getline(linestream, item, ' '))
	nvertices = atoi(item.c_str());
      x = new double[nvertices];
      y = new double[nvertices];
      z = new double[nvertices];
    }
    if (linenum > 2) {
      istringstream linestream(line);
      itemnum = 0;
      ip = linenum - 3;
      while (getline(linestream, item, ' ')) {
 	if (itemnum == 0) x[ip] = atof(item.c_str());
 	if (itemnum == 1) y[ip] = atof(item.c_str());
 	if (itemnum == 2) z[ip] = atof(item.c_str());
	itemnum++;
      }
    }
  }
 inFile.close();

  cout << "reading 3D fields ..." << endl;
  inFile.open("qhullFields.txt"); 
  linenum = 0;
  p = new double[nvertices];
  rho = new double[nvertices];
  while (getline(inFile, line)) {
    linenum++;
    if (linenum > 2) {
      istringstream linestream(line);
      itemnum = 0;
      ip = linenum - 3;
      while (getline(linestream, item, ' ')) {
 	if (itemnum == 0) p[ip] = atof(item.c_str());
 	if (itemnum == 1) rho[ip] = atof(item.c_str());
	itemnum++;
      }
    }
  }
  inFile.close();

  vtkSmartPointer points = vtkSmartPointer::New();
  vtkSmartPointer scalars1 = vtkSmartPointer::New();
  scalars1 -> SetName("pressure");
  vtkSmartPointer scalars2 = vtkSmartPointer::New();
  scalars2 -> SetName("density");

  for (i = 0 ; i < nvertices ; i++) {
    points->InsertNextPoint(x[i],y[i],z[i]);
    scalars1->InsertNextValue(p[i]);
    scalars2->InsertNextValue(rho[i]);
  }

  cout << "writing VTK cells ..." << endl;
  vtkSmartPointer unstructuredGrid = vtkSmartPointer::New();
  unstructuredGrid -> SetPoints(points);
  unstructuredGrid -> GetPointData() -> SetScalars(scalars1);
  unstructuredGrid -> GetPointData() -> AddArray(scalars2);
  for (i = 0 ; i < ncells ; i++) {
    vtkIdType ptIds[] = {cell[i][0], cell[i][1], cell[i][2], cell[i][3]};
    unstructuredGrid->InsertNextCell(VTK_TETRA, 4, ptIds);
    if (i%1000000 == 0) cout << i << " out of " << ncells << endl;
  }

  cout << "writing data to file ..." << endl;
  vtkXMLUnstructuredGridWriter* writer = vtkXMLUnstructuredGridWriter::New();
  writer -> SetInputData(unstructuredGrid);
  writer -> SetFileName("a1.vtu");
  writer -> Write();

  return 0;
}
```

You will need VTK 6.0 or higher. Here is the makefile to compile this code on a cluster:

```
VTKDIR = /global/scratch/razoumov/vtk
coreVTK = -lvtkCommonDataModel-6.2 -lvtkCommonCore-6.2 -lvtkCommonExecutionModel-6.2 \
	-lvtkRenderingCore-6.2 -lvtkRenderingFreeType-6.2 -lvtkfreetype-6.2 \
	-lvtkRenderingOpenGL-6.2 -lvtkInteractionStyle-6.2 -lvtkRenderingFreeTypeOpenGL-6.2 \
	-lvtkIOXML-6.2 -lvtkIOCore-6.2 -lvtkCommonMath-6.2 -lvtkCommonSystem-6.2 -lvtkCommonCore-6.2 \
	-lvtkCommonMisc-6.2 -lvtkCommonTransforms-6.2 -lvtksys-6.2 -lvtkCommonColor-6.2 \
	-lvtkFiltersSources-6.2 -lvtkFiltersGeometry-6.2 -lvtkFiltersExtraction-6.2 \
	-lvtkFiltersGeneral-6.2 -lvtkCommonComputationalGeometry-6.2 -lvtkFiltersCore-6.2 \
	-lvtkFiltersStatistics-6.2 -lvtkImagingFourier-6.2 -lvtkImagingCore-6.2 \
	-lvtkalglib-6.2 -lvtkftgl-6.2 -lvtkzlib-6.2 -lvtkImagingHybrid-6.2 -lvtkIOImage-6.2 \
	-lvtkIOGeometry-6.2 -lvtkIOXMLParser-6.2 -lvtkjpeg-6.2 -lvtkpng-6.2 -lvtktiff-6.2 \
	-lvtkmetaio-6.2 -lvtkDICOMParser-6.2 -lvtkjsoncpp-6.2 -lvtkexpat-6.2
delaunayVTK: delaunayVTK.cpp
	g++ $^ -o $@ -I$(VTKDIR)/include/vtk-6.2 -L$(VTKDIR)/lib $(coreVTK)
clean:
	@/bin/rm -rf *.o *~
	@find . -type f -perm +111 | xargs /bin/rm -rf
```
	
The code will read `qhullTetrahedra.txt`, `qhullParticles.txt` and `qhullFields.txt` and will store the particles and
tessellation in an UnstructuredGrid VTK file called a1.vtu. It assumes that you have two scalar variables per particle
named pressure and density -- you can adapt it to your own data. Import the output VTK file into ParaView, switch to a
volume view, and edit the colour map.

To produce the spin animation seen above, add to your ParaView's Python code:

```
camera = GetActiveCamera()
firstFrame = 0
lastFrame = 90
camera.Azimuth(firstFrame*0.2)
for i in range(lastFrame-firstFrame+1):
    camera.Azimuth(0.2)
    SaveScreenshot(dir+'/frame%04d'%(firstFrame+i)+'.png', magnification=1, quality=100, view=renderView1)
```

where `firstFrame` and `lastFrame` give the start/end positions of the spin in degrees, and 0.2 is the rotation step in
degrees.

## Future work

This single-timestep visualization can be extended to show multiple timesteps, with cosmic structures evolving in
time. The camera position can be adjusted to show fly-throughs, zoom-ins on individual galaxies, etc. Other quantities
such as temperature and metallicity can be added to the visualization. In addition, the algorithm can scale to billions
of particles and beyond, if one uses adaptive tessellation to roughly match the number of resolution elements to the
number of pixels on the screen.

The tessellation artifacts near one of the edges of the box (large red cells), appearing due to a high-density region
located near that edge, can be removed by introducing ghost cells on the outside such that no high-density region is
truncated by the boundary.

## Contact

This visualization was developed by [Alex Razoumov](mailto:alexeir@sfu.ca) (SFU).

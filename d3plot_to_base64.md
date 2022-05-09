```c++
// how to compile in linux
// g++ main.cpp ../lib/liblsreader_cxx.a ../lib/libzipper.a  -I ../include/ -std=c++11 -lz

// Author: Jeff Wang
// this code made use of d3plot reader from LSTC to read thinning values from LsDyna d3plot file
// and then use minizip to compress it to zip.
// the zip file later is turned into base64 string.
// this base64 string will be embededed in a html file.
// in the html file, JsZip.js and Three.js are use to decompress the zip file and visualize it in browser.

#include <string>
#include <iostream>
#include <fstream>
#include <sstream>
#include <iomanip>
#include <vector>
#include "config.h"
#include "data_zipper.h"
#include "d3reader-wrapper.h"

using namespace std;

vector<float> extract_intensity(D3P_Shell *shells, float *shell_thickness, size_t num_elems)
{
    int index0, index1, index2, index3;

    // each elements can be break to
    // 2 triangle,
    // each triangle has 3 nodes.
    // each node has X, Y, Z components.
    // need to initiate to 0, added () at end;
    // this part is really confusing.
    vector<float> result;

    // if triangle element, use node 0,1,2
    for (size_t i = 0; i < num_elems; i++)
    {
        result.push_back(shell_thickness[i]);
        result.push_back(shell_thickness[i]);
        result.push_back(shell_thickness[i]);

        index2 = shells[i].conn[2];
        index3 = shells[i].conn[3];

        if (index2 != index3)
        {
            result.push_back(shell_thickness[i]);
            result.push_back(shell_thickness[i]);
            result.push_back(shell_thickness[i]);
        }
    }
    return result;
}

vector<float> extract_mesh(D3P_Shell *shells, int *nodal_ids, D3P_Vector *coords, size_t num_elems)
{
    int index0, index1, index2, index3;

    // each elements can be break to
    // 2 triangle,
    // each triangle has 3 nodes.
    // each node has X, Y, Z components.
    // need to initiate to 0, added () at end;
    // this part is really confusing.
    vector<float> result;

    // if triangle element, use node 0,1,2
    for (size_t i = 0; i < num_elems; i++)
    {
        // index0 = nodal_ids[shells[i].conn[0] - 1];
        index0 = shells[i].conn[0] - 1;
        result.push_back(coords[index0].v[0]);
        result.push_back(coords[index0].v[1]);
        result.push_back(coords[index0].v[2]);

        // index1 = nodal_ids[shells[i].conn[1] - 1];
        index1 = shells[i].conn[1] - 1;
        result.push_back(coords[index1].v[0]);
        result.push_back(coords[index1].v[1]);
        result.push_back(coords[index1].v[2]);

        // index2 = nodal_ids[shells[i].conn[2] - 1];
        index2 = shells[i].conn[2] - 1;
        result.push_back(coords[index2].v[0]);
        result.push_back(coords[index2].v[1]);
        result.push_back(coords[index2].v[2]);

        // deal with quad element, use node 0,2,3
        // index3 = nodal_ids[shells[i].conn[3] - 1];
        index3 = shells[i].conn[3] - 1;

        /*
        printf("%d\n", i);
        printf("%d %d %d %d\n", nodal_ids[index0], nodal_ids[index1], nodal_ids[index2], nodal_ids[index3]);
        printf("%f %f %f\n", coords[index0].v[0], coords[index0].v[1], coords[index0].v[2]);
        printf("%f %f %f\n", coords[index1].v[0], coords[index1].v[1], coords[index1].v[2]);
        printf("%f %f %f\n", coords[index2].v[0], coords[index2].v[1], coords[index2].v[2]);
        printf("%f %f %f\n\n", coords[index3].v[0], coords[index3].v[1], coords[index3].v[2]);
        */

        if (index2 != index3)
        {
            result.push_back(coords[index0].v[0]);
            result.push_back(coords[index0].v[1]);
            result.push_back(coords[index0].v[2]);
            result.push_back(coords[index2].v[0]);
            result.push_back(coords[index2].v[1]);
            result.push_back(coords[index2].v[2]);
            result.push_back(coords[index3].v[0]);
            result.push_back(coords[index3].v[1]);
            result.push_back(coords[index3].v[2]);
        }
    }
    return result;
}

// return JSON string.
string extract_text(float *shellsThickness, size_t num_elems)
{
    // string vector to JSON string
    std::ostringstream ostr;
    ostr << '[' << std::endl; // leading half bracket
    size_t i;
    // add data one by one, skip last element
    for (i = 0; i < num_elems - 1; i++)
    {
        ostr << "\"" << std::to_string(shellsThickness[i]) << "\",";
    }
    // last element no ending ","
    ostr << "\"" << std::to_string(shellsThickness[i]) << "\"";
    ostr << ']';
    return ostr.str();
}

// todo: 1. export each step as separate folder inside zip file.
// todo: 2. probably need to use multi thread to improve efficiency.
int main()
{
    string d3plot_file = DATA_PATH_D3P;

    D3plotReader dr(d3plot_file.c_str());

    cout << "=============== D3plotReader ===============" << endl;
    char title[40];
    dr.GetData(D3P_TITLE, (char *)title);
    cout << "title: " << title << endl;

    // get number of states
    int num_states = 0;
    dr.GetData(D3P_NUM_STATES, (char *)&num_states);

    // get node coordinates, last step.
    D3P_Parameter param;
    D3P_Init(&param); // this step is necessary for ls reader C interface.
    param.ipart = 0;

    // first step.
    param.ist = 0;

    // get number of nodes.
    int num_nodes = 0;
    dr.GetData(D3P_NUM_NODES, (char *)&num_nodes, param);

    // get number of elements.
    int num_elems = 0;
    dr.GetData(D3P_NUM_SHELL, (char *)&num_elems, param);

    // get element connectivity
    D3P_Shell *shells = new D3P_Shell[num_elems];
    dr.GetData(D3P_SHELL_CONNECTIVITY_MAT, (char *)shells, param);

    // get nodal_id
    int *nodal_ids = new int[num_nodes];
    dr.GetData(D3P_NODE_IDS, (char *)nodal_ids, param);

    data_zipper zip1("test.zip");

    D3P_Vector *coords = new D3P_Vector[num_nodes];
    float *shell_thickness = new float[num_elems];

	// zip each state as separate entry in zip archive.
	// the result zip file size can be huge. 
    for(int i = 0; i <= num_states; i++	)
    {
        // show status
        cout <<"***************working on state " << i << " ***************\n";

        // create folder for each step.
        string folder = std::to_string(i) + "/";

        param.ist = i;

        dr.GetData(D3P_NODE_COORDINATES, (char *)coords, param);

        // get thickness information.

        dr.GetData(D3P_SHELL_THICKNESS, (char *)shell_thickness, param);

        // use vector as container. get geometry information
        // As of C++11, you can also use the str.data() member function, which returns char *
        // char *cstr = &str[0];. two ways to use string in place of char *
        vector<float> resultGeo = extract_mesh(shells, nodal_ids, coords, num_elems);
        zip1.zip_file(resultGeo, (folder + "positions").data());

        // get shell thickness information. simply convert array to vector using vector constructor.
        vector<float> resultThickness = extract_intensity(shells, shell_thickness, num_elems);
        zip1.zip_file(resultThickness, (folder + "intensities").data());

        // get element value to used as tool tip. need to convert string array to JSON.
        string resultText = extract_text(shell_thickness, num_elems);
        // add begin and end []
        zip1.zip_file(resultText, (folder + "text").data());
    }

    // call destructor to close zip. Closing process will also write some data to end of zip.
    // if you work on the zip file before close it, the file will be corrupted. 7zip is still able to read it
    // 7zip will give a warning message. Windows explorer or JSZip will not be able to read the corrupted zip file.
    zip1.close_zip();

    zip1.export_zip_as_base64("test.zip", "testBase64");

    delete[] coords;
    delete[] shell_thickness;
    delete[] shells;
    delete[] nodal_ids;
    return 0;
}

```

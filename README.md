  RPly: ANSI C library for PLY file format input and output  

- - -

|     |
| --- |
| ![RPly](rply.png) |
| ANSI C Library for PLY file format input and output |

- - -

# Introduction

RPly is a library that lets applications read and write PLY files. The PLY file format is widely used to store geometric information, such as 3D models, but is general enough to be useful for other purposes.

There are other libraries out there, of course. I tried using them and finally decided to write my own. The result is RPly, and I hope you are as happy with it as I am.

RPly is easy to use, well documented, small, free, open-source, ANSI C, efficient, and well tested. I will keep supporting it for a while because all my tools use the library for input/output. The highlights are:

*   A callback mechanism that makes PLY file input straightforward;
*   Support for the full range of numeric formats though the user only deals with doubles;
*   Binary (big and little endian) and text modes are fully supported;
*   Input and output are buffered for efficiency;
*   Available under the [MIT license](http://www.opensource.org/licenses/mit-license.html) for added freedom.

The format was developed at [Stanford University](http://graphics.stanford.edu/data/3Dscanrep/) for use with their 3D scanning projects. Greg Turk's PLY library, available from [Georgia Institute of Technology](http://www.cc.gatech.edu/projects/large_models), seems to be the standard reference to the PLY file format, although there are some variations out there.

Whatever documentation and examples were found, were taken into consideration to create RPly. In theory, since RPly doesn't try to interpret the meaning of the data in a PLY file, it should be able to read any PLY file. In practice, the library works with all PLY files that I could find.

## Download

Version 1.1.4 of RPly is available for download in source code from [this link](rply-1.1.4.tar.gz). Examples and documentation are packed inside the tarball. Have fun!

Copyright © 2003-2015 Diego Nehab. All rights reserved.  
Author: [Diego Nehab](http://www.impa.br/~diego)

## What's new?

*   Fixed bug that prevented reading of ASCII files that are not terminated by a whitespace;
*   Added ply\_open\_from\_file and ply\_create\_to\_file variants to ply\_open and ply\_create, respectively, that receive file pointers instead of file names.

## RPly's idea of what a PLY file is

A PLY file contains the description of one object. This object is composed by _elements_, each element type being defined by a group of _properties_. The PLY file format specifies a syntax for the description of element types and the properties that compose them, as well as comments and meta-information.

The element type descriptions come in a header, which is followed by element instances. Element instances come grouped by their type, in the order of declaration. Each element instance is defined by the value of its properties. Properties values also appear in the order of their declaration. Here is a sample PLY file describing a triangle:

ply
format ascii 1.0
comment this is a simple file
obj\_info any data, in one line of free form text
element vertex 3
property float x
property float y
property float z
element face 1
property list uchar int vertex\_indices
end\_header
-1 0 0
 0 1 0
 1 0 0
3 0 1 2

The header goes from the first line to the line marked by end\_header. The first line contains only ply\\n and is used to detect whether a file is in PLY format or not (RPly also accepts files that start with ply\\r\\n, in which case the end-of-line terminator is assumed to be \\r\\n throughout.) The second line specifies the format number (which is always 1.0) and the storage mode (ascii, binary\_big\_endian or binary\_little\_endian).

Lines that start with comment are just comments, of course. Lines that start with obj\_info contain meta-information about the object. Comments and obj\_infos are optional and their relative order in the header is irrelevant.

In the sample PLY file, the first element type is declared with name vertex, and on the same line we learn that there will be 3 instances of this element type. The properties following describe what a vertex element looks like. Each vertex is declared to consist of 3 scalar properties, named x, y and z. Each scalar property is declared to be of type float.

Scalar types can be any of the following: int8, uint8, int16, uint16, int32, uint32, float32, float64, char, uchar, short, ushort, int, uint, float, double. They consist of signed and unsigned integer types of sizes 8, 16 and 32 bits, as well as floating point types of 32 and 64bits.

Next, the face element type is declared, of which only 1 instance will be given. This element consists of a list property, named vertex\_indices. Lists are sequences on which the first value, the _length_, gives the number of remaining values. List properties are described by the scalar type of their length field and the scalar type of the remaining fields. In the case of vertex\_indices, the length field is of type uchar and the remaining values are of type int.

Following the header, come the elements, in the order they were declared in the header. First come the 3 elements of type vertex, each represented by the value of their properties x, y and z. Then comes the single face element, composed by a single list of type vertex\_indices containing 3 values (0 1 2).

## How to read a file with RPly

Most users that want to read a PLY file already know which elements and properties they are interested in. In the following example, we will implement a simple program that dumps the contents of a PLY file to the terminal, in a different, simpler format that only works for triangles.

This simple format has a header that gives the number of vertices in the first line and the number of triangles in the second line. Following the header come the vertices, and finally the triangles. Here is the sample code for the program:

#include <stdio.h>
#include "rply.h"

static int vertex\_cb(p\_ply\_argument argument) {
    long eol;
    ply\_get\_argument\_user\_data(argument, NULL, &eol);
    printf("%g", ply\_get\_argument\_value(argument));
    if (eol) printf("\\n");
    else printf(" ");
    return 1;
}

static int face\_cb(p\_ply\_argument argument) {
    long length, value\_index;
    ply\_get\_argument\_property(argument, NULL, &length, &value\_index);
    switch (value\_index) {
        case 0:
        case 1:
            printf("%g ", ply\_get\_argument\_value(argument));
            break;
        case 2:
            printf("%g\\n", ply\_get\_argument\_value(argument));
            break;
        default:
            break;
    }
    return 1;
}

int main(void) {
    long nvertices, ntriangles;
    p\_ply ply = ply\_open("input.ply", NULL, 0, NULL);
    if (!ply) return 1;
    if (!ply\_read\_header(ply)) return 1;
    nvertices = ply\_set\_read\_cb(ply, "vertex", "x", vertex\_cb, NULL, 0);
    ply\_set\_read\_cb(ply, "vertex", "y", vertex\_cb, NULL, 0);
    ply\_set\_read\_cb(ply, "vertex", "z", vertex\_cb, NULL, 1);
    ntriangles = ply\_set\_read\_cb(ply, "face", "vertex\_indices", face\_cb, NULL, 0);
    printf("%ld\\n%ld\\n", nvertices, ntriangles);
    if (!ply\_read(ply)) return 1;
    ply\_close(ply);
    return 0;
}

RPly uses callbacks to pass data to an application. Independent callbacks can be associated with each property of each element. For scalar properties, the callback is invoked once for each instance. For list properties, the callback is invoked first with the number of entries in the instance, and then once for each of the data entries. _This is exactly the order in which the data items appear in the file._

To keep things simple, values are always passed as double, regardless of how they are stored in the file. From its parameters, callbacks can find out exactly which part of the file is being processed (including the actual type of the value), plus access custom information provided by the user in the form of a pointer and an integer constant.

In our example, we start with a call to ply\_open to open a file for reading. Then we get RPly to parse it's header, with a call to ply\_read\_header. After the header is parsed, RPly knows which element types and properties are available. We then set callbacks for each of the vertex element properties and the face property (using ply\_set\_read\_cb). Finally, we invoke the main RPly reading function, ply\_read. This function reads all data in the file, passing the data to the appropriate callbacks. After all reading is done, we call ply\_close to release any resources used by RPly.

There are some details, of course. Ply\_set\_read\_cb returns the number of instances of the target property (which is the same as the number of element instances). This is how the program obtains the number of vertices and faces in the file.

RPly lets us associate one pointer _and_ one integer to each callback. We are free to use either or both to link some context to our callbacks. Our example uses the integer placeholder to tell vertex\_cb that it has to break the line after the z property (notice the last argument of ply\_set\_read\_cb).

Vertex\_cb gets the user data and the property value from it's argument and prints accordingly. The face\_cb callback is a bit more complicated because lists are more complicated. Since the simple file format only supports triangles, it only prints the first 3 list values, after which it breaks the line.

The output of the program, as expected, is:

3
1
-1 0 0
0 1 0
1 0 0
0 1 2

## Writing files with RPly

The next example is somewhat more involved. We will create a program that converts our simple PLY file to binary mode. Besides showing how to write a PLY file, this example also illustrates the query functions. We do not know a priori which elements and properties, comments and obj\_infos will be in the input file, so we need a way to find out. Although our simple program would work on any PLY file, a better version of this program is available from the RPly distribution. For simplicity, the simple version omits error messages and command line parameter processing.

In practice, writing a file is even easier than reading one. First we create a file in binary mode, with a call to ply\_create (notice the argument PLY\_LITTLE\_ENDIAN that gives the storage mode). Then, we define the elements using ply\_add\_element. After each element, we define its properties using ply\_add\_scalar\_property or ply\_add\_list\_property. When we are done with elements and properties, we add comments and obj\_infos. We then write the header with ply\_write\_header and send all data items. The data items are sent one by one, with calls to ply\_write, _in the same order they are to appear in the file_. Again, to simplify things, this function receives data as double and performs the needed conversion. Here is the code for the example:

#include <stdio.h>
#include "rply.h"

static int callback(p\_ply\_argument argument) {
    void \*pdata;
    /\* just pass the value from the input file to the output file \*/
    ply\_get\_argument\_user\_data(argument, &pdata, NULL);
    ply\_write((p\_ply) pdata, ply\_get\_argument\_value(argument));
    return 1;
}

static int setup\_callbacks(p\_ply iply, p\_ply oply) {
    p\_ply\_element element = NULL;
    /\* iterate over all elements in input file \*/
    while ((element = ply\_get\_next\_element(iply, element))) {
        p\_ply\_property property = NULL;
        long ninstances = 0;
        const char \*element\_name;
        ply\_get\_element\_info(element, &element\_name, &ninstances);
        /\* add this element to output file \*/
        if (!ply\_add\_element(oply, element\_name, ninstances)) return 0;
        /\* iterate over all properties of current element \*/
        while ((property = ply\_get\_next\_property(element, property))) {
            const char \*property\_name;
            e\_ply\_type type, length\_type, value\_type;
            ply\_get\_property\_info(property, &property\_name, &type,
                    &length\_type, &value\_type);
            /\* setup input callback for this property \*/
            if (!ply\_set\_read\_cb(iply, element\_name, property\_name, callback,
                    oply, 0)) return 0;
            /\* add this property to output file \*/
            if (!ply\_add\_property(oply, property\_name, type, length\_type,
                    value\_type)) return 0;
        }
    }
    return 1;
}

int main(int argc, char \*argv\[\]) {
    const char \*value;
    p\_ply iply, oply;
    iply = ply\_open("input.ply", NULL, 0, NULL);
    if (!iply) return 1;
    if (!ply\_read\_header(iply)) return 1;
    oply = ply\_create("output.ply", PLY\_LITTLE\_ENDIAN, NULL, 0, NULL);
    if (!oply) return 1;
    if (!setup\_callbacks(iply, oply)) return 1;
    /\* pass comments and obj\_infos from input to output \*/
    value = NULL;
    while ((value = ply\_get\_next\_comment(iply, value)))
        if (!ply\_add\_comment(oply, value)) return 1;
    value = NULL;
    while ((value = ply\_get\_next\_obj\_info(iply, value)))
        if (!ply\_add\_obj\_info(oply, value)) return 1;;
    /\* write output header \*/
    if (!ply\_write\_header(oply)) return 1;
    /\* read input file generating callbacks that pass data to output file \*/
    if (!ply\_read(iply)) return 1;
    /\* close up, we are done \*/
    if (!ply\_close(iply)) return 1;
    if (!ply\_close(oply)) return 1;
    return 0;
}

RPly uses iterators to let the user loop over a PLY file header. A function is used to get the first item of a given class (element, property etc). Passing the last returned item to the same function produces the next item, until there are no more items. Examples of iterator use can be seen in the main function, which uses them to loop over comments and obj\_infos, and in the setup\_callbacks function, which loops over elements and properties.

In the setup\_callbacks function, for each element in the input, an equivalent element is defined in the output. For each property in each element, an equivalent property is defined in the output. Notice that the same callback is specified for all properties. It is given the output PLY handle as the context pointer. Each time it is called, it passes the received value to ply\_write on the output handle. It is as simple as that.

## A note on locale

ASCII PLY files are supposed to use the C locale for numeric formatting. RPly relies on library functions (such as fprintf and strtod) that are affected by the current locale. If your software modifies the locale (or if it uses another library/toolkit that does) and you use RPly under the modified locale, you may be unable to read or write properly formatted ASCII PLY files.

Modifying RPly internally to hedge against different locales would be complicated, particularly in multi-threaded applications. Therefore, RPly leaves this as your responsibility. To protect against locale problems in the simplest scenario, you should bracket RPly I/O as follows:

#include <locale.h>
/\* Save application locale \*/
const char \*old\_locale = setlocale(LC\_NUMERIC, NULL);
/\* Change to PLY standard \*/
setlocale(LC\_NUMERIC, "C");
/\* Use the RPly library \*/
...

/\* Restore application locale when done \*/
setlocale(LC\_NUMERIC, old\_locale);

# Reference Manual

p\_ply **ply\_open**(const char \*name, p\_ply\_error\_cb error\_cb, long idata, void \*pdata)

Opens a PLY file for reading, checks if it is a valid PLY file and returns a handle to it.

Name is the file name, and error\_cb is a function to be called when an error is found. Arguments idata and pdata are available to the error callback via the [ply\_get\_ply\_user\_data](#ply_get_ply_user_data) function. If error\_cb is NULL, the default error callback is used. It prints a message to the standard error stream.

Returns a handle to the file or NULL on error.

Note: Error\_cb is of type void (\*p\_ply\_error\_cb)(p\_ply ply, const char \*message).

p\_ply **ply\_open\_from\_file**(FILE \*file\_pointer, p\_ply\_error\_cb error\_cb, long idata, void \*pdata)

Checks if the FILE pointer points to a valid PLY file and returns a handle to it. The handle can be used wherever a handle returned by [ply\_open](#ply_open) is accepted.

File\_pointer is the FILE pointer open for reading, and error\_cb is a function to be called when an error is found. Arguments idata and pdata are available to the error callback via the [ply\_get\_ply\_user\_data](#ply_get_ply_user_data) function. If error\_cb is NULL, the default error callback is used. It prints a message to the standard error stream.

Returns a handle to the file or NULL on error.

Note: Error\_cb is of type void (\*p\_ply\_error\_cb)(p\_ply ply, const char \*message).

Note: This function is declared in header rplyfile.h.

int **ply\_get\_ply\_user\_data**(p\_ply\_ply ply, void \*pdata, long \*idata)

Retrieves user data from the ply handle.

Ply is the handle passed to the error callback. Pdata receives the user data pointer. Idata receives the user data integer. Pdata and idata can be NULL.

Returns 1 in case of success, 0 otherwise.

int **ply\_read\_header**(p\_ply ply)

Reads and parses the header of a PLY file. After a call to this function, the query functions [ply\_get\_next\_element](#ply_get_next_element), [ply\_get\_next\_property](#ply_get_next_property), [ply\_get\_next\_comment](#ply_get_next_comment), and [ply\_get\_next\_obj\_info](#ply_get_next_obj_info) can be called. Callbacks can also be set with the [ply\_set\_read\_cb](#ply_set_read_cb) function.

Ply is a handle returned by [ply\_open](#ply_open).

Returns 1 in case of success, 0 otherwise.

long **ply\_set\_read\_cb**(  
   p\_ply ply,  
   const char \*element\_name,  
   const char \*property\_name,  
   p\_ply\_read\_cb read\_cb,  
   void \*pdata,  
   long idata  
)

Sets up the callback to be invoked when the value of a property is read.

Ply is a handle returned by [ply\_open](#ply_open). Element\_name and property\_name are the names of the element and property of interest. Read\_cb is the callback function. Pdata and idata are user data to be passed to the callback function.

Returns the number of instances of the element of interest.

Note: Read\_cb is of type int (\*p\_ply\_read\_cb)(p\_ply\_argument argument). The callback should return 1 to continue the reading process, or return 0 to abort.

int **ply\_get\_argument\_element**(  
   p\_ply\_argument argument,  
   p\_ply\_element \*element,  
   long \*instance\_index  
)

Retrieves element information from the callback argument.

Argument is the handle passed to the callback. Element receives a handle to the element originating the callback. Instance\_index receives the index of the instance of the element being read. Element and instance\_index can be NULL.

Returns 1 in case of success, 0 otherwise.

Note: further information can be obtained from element with a call to [ply\_get\_element\_info](#ply_get_element_info).

int **ply\_get\_argument\_property**(  
   p\_ply\_argument argument,  
   p\_ply\_property \*property,  
   long \*length,  
   long \*value\_index  
)

Retrieves property information from the callback argument.

Argument is the handle passed to the callback. Property receives a handle to the property originating the callback. Length receives the number of values in the list property (1 for scalar properties). Value\_index receives the index of the current property entry (0 for scalar properties, -1 for the first value of a list property, the one that gives the number of entries). Property, length and value\_index can be NULL.

Returns 1 in case of success, 0 otherwise.

Note: further information can be obtained from property with a call to [ply\_get\_property\_info](#ply_get_property_info).

int **ply\_get\_argument\_user\_data**(p\_ply\_argument argument, void \*pdata, long \*idata)

Retrieves the user data from the callback argument.

Argument is the handle passed to the callback. Pdata receives the user data pointer. Idata receives the user data integer. Pdata and idata can be NULL.

Returns 1 in case of success, 0 otherwise.

double **ply\_get\_argument\_value**(p\_ply\_argument argument)

Retrieves the property value from the callback argument.

Argument is the handle passed to the callback.

Returns the property value.

int **ply\_read**(p\_ply ply)

Reads all data in file, calling appropriate callbacks.

Ply is a handle returned by [ply\_open](#ply_open).

Returns 1 in case of success, 0 otherwise.

p\_ply\_element **ply\_get\_next\_element**(p\_ply ply, p\_ply\_element last)

Iterates over all elements on the header of a PLY file.

Ply is a handle returned by [ply\_open](#ply_open). [Ply\_read\_header](#ply_read_header) must have been called on the handle otherwise no elements will be found. Last is NULL to retrieve the first element, and an element to retrieve the next element.

Returns the next element, or NULL if no more elements.

Note: further information can be obtained from an element with a call to [ply\_get\_element\_info](#ply_get_element_info).

p\_ply\_property **ply\_get\_next\_property**(p\_ply\_element element, p\_ply\_property last)

Iterates over all properties of an element.

Element is an element handle. Last is NULL to retrieve the first property, and a property to retrieve the next property.

Returns the next property, or NULL if no more properties.

Note: further information can be obtained from a property with a call to [ply\_get\_property\_info](#ply_get_property_info).

const char \***ply\_get\_next\_comment**(p\_ply ply, const char \*last)

Iterates over all comments on the header of a PLY file.

Ply is a handle returned by [ply\_open](#ply_open). [Ply\_read\_header](#ply_read_header) must have been called on the handle otherwise no comments will be found. Last is NULL to retrieve the first comment, and a comment to retrieve the next comment.

Returns the next comment, or NULL if no more comments.

const char \***ply\_get\_next\_obj\_info**(p\_ply ply, const char \*last)

Iterates over all obj\_infos on the header of a PLY file.

Ply is a handle returned by [ply\_open](#ply_open). [Ply\_read\_header](#ply_read_header) must have been called on the handle otherwise no obj\_infos will be found. Last is NULL to retrieve the first obj\_info, and a obj\_info to retrieve the next obj\_info.

Returns the next obj\_info, or NULL if no more obj\_infos.

int **ply\_get\_element\_info**(p\_ply\_element element, const char\*\* name, long \*ninstances)

Retrieves information from an element handle.

Element is the handle of the element of interest. Name receives the internal copy of the element name. Ninstances receives the number of instances of this element in the file. Both name and ninstances can be NULL.

Returns 1 in case of success, 0 otherwise.

int **ply\_get\_property\_info**(  
   p\_ply\_property property,  
   const char\*\* name,  
   e\_ply\_type \*type,  
   e\_ply\_type \*length\_type,  
   e\_ply\_type \*value\_type  
)

Retrieves information from a property handle.

Property is the handle of the property of interest. Name receives the internal copy of the property name. Type receives the property type. Length\_type receives the scalar type of the first entry in a list property (the one that gives the number of entries). Value\_type receives the scalar type of the remaining list entries. Name, type, length\_type, and value\_type can be NULL.

Returns 1 in case of success, 0 otherwise.

Note: Length\_type and value\_type can receive any of the constants for scalar types defined in e\_ply\_type. Type can, in addition, be PLY\_LIST, in which case the property is a list property and the fields length\_type and value\_type become meaningful.

p\_ply **ply\_create**(const char \*name, e\_ply\_storage\_mode storage\_mode, p\_ply\_error\_cb error\_cb)

Creates a PLY file for writing.

Name is the file name, storage\_mode is the file storage mode (PLY\_ASCII, PLY\_LITTLE\_ENDIAN, PLY\_BIG\_ENDIAN, or PLY\_DEFAULT to automatically detect host endianess). Error\_cb is a function to be called when an error is found. Arguments idata and pdata are available to the error callback via the [ply\_get\_ply\_user\_data](#ply_get_ply_user_data) function. If error\_cb is NULL, the default error callback is used. It prints a message to the standard error stream.

Returns a handle to the file or NULL on error.

Note: Error\_cb is of type void (\*p\_ply\_error\_cb)(const char \*message)

p\_ply **ply\_create\_to\_file**(FILE \*file\_pointer, e\_ply\_storage\_mode storage\_mode, p\_ply\_error\_cb error\_cb)

Creates a PLY file to be written to a FILE pointer and returns a handle to it. The handle can be used wherever a handle returned by [ply\_create](#ply_create) is accepted.

File\_pointer a pointer to a file open for writing, storage\_mode is the file storage mode (PLY\_ASCII, PLY\_LITTLE\_ENDIAN, PLY\_BIG\_ENDIAN, or PLY\_DEFAULT to automatically detect host endianess). Error\_cb is a function to be called when an error is found. Arguments idata and pdata are available to the error callback via the [ply\_get\_ply\_user\_data](#ply_get_ply_user_data) function. If error\_cb is NULL, the default error callback is used. It prints a message to the standard error stream.

Returns a handle to the file or NULL on error.

Note: Error\_cb is of type void (\*p\_ply\_error\_cb)(const char \*message)

Note: This function is declared in header rplyfile.h.

int **ply\_add\_element**(p\_ply ply, const char \*name, long ninstances)

Adds a new element to the ply file.

Ply is a handle returned by [ply\_create](#ply_create), name is the element name and ninstances is the number of instances of this element that will be written to the file.

Returns 1 in case of success, 0 otherwise.

int **ply\_add\_property**(  
   p\_ply ply,  
   const char \*name,  
   e\_ply\_type type,  
   e\_ply\_type length\_type,  
   e\_ply\_type value\_type  
)

Adds a new property to the last element added to the ply file.

Ply is a handle returned by [ply\_create](#ply_create) and name is the property name. Type is the property type. Length\_type is the scalar type of the first entry in a list property (the one that gives the number of entries). Value\_type is the scalar type of the remaining list entries. If type is not PLY\_LIST, length\_type and value\_type are ignored.

Returns 1 in case of success, 0 otherwise.

Note: Length\_type and value\_type can be any of the constants for scalar types defined in e\_ply\_type. Type can, in addition, be PLY\_LIST, in which case the property is a list property and the fields length\_type and value\_type become meaningful.

int **ply\_add\_list\_property**(  
   p\_ply ply,  
   const char \*name,  
   e\_ply\_type length\_type,  
   e\_ply\_type value\_type  
)

Same as [ply\_add\_property](#ply_add_property) if type is PLY\_LIST.

int **ply\_add\_scalar\_property**(p\_ply ply, const char \*name, e\_ply\_type type)

Same as [ply\_add\_property](#ply_add_property) if type is _not_ PLY\_LIST.

int **ply\_add\_comment**(p\_ply ply, const char \*comment);

Adds a comment to a PLY file.

Ply is a handle returned by [ply\_create](#ply_create) and comment is the comment text.

Returns 1 in case of success, 0 otherwise.

int **ply\_add\_obj\_info**(p\_ply ply, const char \*obj\_info);

Adds a obj\_info to a PLY file.

Ply is a handle returned by [ply\_create](#ply_create) and obj\_info is the obj\_info text.

Returns 1 in case of success, 0 otherwise.

int **ply\_write\_header**(p\_ply ply);

Writes the PLY file header to disk, after all elements, properties, comments and obj\_infos have been added to the handle.

Ply is a handle returned by [ply\_create](#ply_create) and comment is the comment text.

Returns 1 in case of success, 0 otherwise.

int **ply\_write**(p\_ply ply, double value);

Passes a value to be stored in the PLY file. Values must be passed in the order they will appear in the file.

Ply is a handle returned by [ply\_create](#ply_create) and value is the value to be stored. For simplicity, values are always passed as double and conversion is performed as needed.

Returns 1 in case of success, 0 otherwise.

int **ply\_close**(p\_ply ply);

Closes the handle and ensures that all resources have been freed and data have been written.

Ply is a handle returned by [ply\_create](#ply_create) or by [ply\_open](#ply_open).

Returns 1 in case of success, 0 otherwise.

- - -

Documentation last modified by Diego Nehab on Fri Aug 21 17:11:09 BRT 2015

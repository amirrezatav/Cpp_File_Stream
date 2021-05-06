# Reading and writing data from/to binary files
Some of the data programs work with has to be persisted to disk files in various ways, that can include storing it in a database or to flat files, either as text or binary data.
This recipe is focused on persisting and loading both raw data and objects from and to binary files. In this context, raw data means unstructured data, and in this recipe, we will consider writing and reading the content of a buffer (that is, a contiguous sequence of memory, that can be either a C-like array, an `std::vector`, or an `std::array`).

## Reading and writing **raw** data from/to binary files
For this recipe, you should be familiar with the standard stream input/output library, though some explanations, to the extent required to understand this recipe, are provided below. You should also be familiar with the difference between binary and text files.
and [See This Slides]()
### How to do it...
> ### To write the content of a buffer (in our example, an `std::vector`) to a binary file, you should perform the following steps:
0. Creat your vector 

        std::vector<unsigned char> output {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
1. Open a file stream for writing in binary mode by creating an instance of the `std::ofstream` class

        std::ofstream ofile("sample.bin", std::ios::binary);
2. Ensure that the file is actually open before writing data to the file:

       if(ofile.is_open())
        {
        // streamed file operations
        }
3. Write the data to the file by providing a pointer to the array of characters and the number of characters to write:

       ofile.write(reinterpret_cast<char*>(output.data()), output.size());
       // or ofile.write((char*)output.data(), output.size());
4. Flush the content of the stream buffer to the actual disk file; this isautomatically done when you close the stream:

        ofile.close();
> ### To read the entire content of a binary file to a buffer, you should perform the following steps:
1. Open a file stream to read from a file in the binary mode by creating an instance of the std::ifstream class:

        std::ifstream ifile("sample.bin", std::ios::binary);
2. Ensure that the file is actually opened before reading data from it:

        if(ifile.is_open())
         {
          // streamed file operations
         }
3. Determine the length of the file by positioning the input position indicator to the end of the file, read its value, and then move the indicator to the beginning:

        ifile.seekg(0, std::ios_base::end);
        auto length = ifile.tellg();
        ifile.seekg(0, std::ios_base::beg);
4. Allocate memory to read the content of the file: 
 
       std::vector<unsigned char> input;
       input.resize(static_cast<size_t>(length));
5. Read the content of the file to the allocated buffer by providing a pointer to the array of characters for receiving the data and the number of characters to read:

        ifile.read(reinterpret_cast<char*>(input.data()), length);
6. Check that the read operation is completed successfully:

       auto success = !ifile.fail() && length == ifile.gcount();
7. Finally, close the file stream:

        ifile.close();
## The example code discussed so far in this recipe can be reorganized in the form of two general functions for writing and reading data to and from a file:

        #include <iostream>
        #include <string>
        #include <fstream>
        #include <functional>
        using namespace std;
        bool write_data(char const * const filename, char const * const data, size_t const size)
        {
        auto success = false;
        ofstream ofile(filename, std::ios::binary);
        if(ofile.is_open())
        {
                try
                {
                   ofile.write(data, size);
                  success = true;
                }
                catch(std::ios_base::failure &)
                {
                // handle the error
                }
                ofile.close();
        }
        return success;
        }
        size_t read_data(char const * const filename, std::function<char*(size_t const)> allocator)
        {
        size_t readbytes = 0;
        std::ifstream ifile(filename, std::ios::ate | std::ios::binary);
        if(ifile.is_open())
        {
        auto length = static_cast<size_t>(ifile.tellg());
        ifile.seekg(0, std::ios_base::beg);
        auto buffer = allocator(length);
                try
                {
                ifile.read(buffer, length);
                readbytes = static_cast<size_t>(ifile.gcount());
                }
                catch (std::ios_base::failure &)
                {
                // handle the error
                }
                ifile.close();
        }
        return readbytes;
        }
        #include<vector>
        int main()
        {
        std::vector<unsigned char> output{ 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
        std::vector<unsigned char> input;
        if (write_data("sample.bin",(char*)output.data(),output.size()))
        {
                if (read_data("sample.bin",[&input](size_t const length) {input.resize(length);return (char*)(input.data()); }) > 0)
                {
                std::cout << (output == input ? "equal" : "not equal") << std::endl;
                }
        }
        }

# Reading and writing objects from/to binary files
In the previous recipe, we saw how to write and read raw data (that is, unstructured data) to and from a file. Many times, however, we have to persist and load objects. Writing and reading in the manner shown in the previous recipe works for POD types only. For anything else, we must explicitly decide what is actually written or read, as writing or reading pointers, virtual tables, and any sort of meta data is not only irrelevant, but also semantically wrong. These operations are commonly referred to as serialization and deserialization. In this recipe, we will see how we can serialize and deserialize both POD and non-POD types to and from binary files.

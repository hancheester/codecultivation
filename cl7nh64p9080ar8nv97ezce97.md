## A note of caution on GCC’s uninitialized variables

Recently, I have started my journey to learn C programming using Visual Studio Code and Mingw-w64. What is [Mingw-w64](http://mingw-w64.org)? It is created to support GCC compiler on Windows systems. [GCC](https://gcc.gnu.org/) is the GNU Compiler Collection and GNU (stands for GNU’s not Unix) is key component to help build operating systems like Linux.

If you are interested to find out how to use Mingw-w64 in VS Code, please click this [link](https://code.visualstudio.com/docs/cpp/config-mingw) to start.

Once everything is setup, you should have 3 files in .vscode sub-folder:

*   c\_cpp\_properties.json (compiler path and IntelliSense settings)
*   tasks.json (build instructions)
*   launch.json (debugger settings)

Here is the file program.c which I wrote to check the variables.

    #include<stdio.h>
    
    int main()
    {
        int apples;
        int oranges;
        int bananas;
    
        printf("apples=%d oranges=%d bananas=%d", apples, oranges, bananas);
    
        return 0;
    }

So, if we look at the codes, all those variables are not initialized. Meaning those variables have no value assigned to them. So, if those variables are not initialized, they should have default value which in this case, they all should have default value 0. So I would expect to have all those variables printed as 0.

If you have setup your environment properly, you should be able to run the command below:

`gcc program.c -o program`

Once executed, simply run program.exe.

But look at my output screenshot below:

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286806475/qLloyxPLy.png)](https://codecultivation.files.wordpress.com/2020/11/image.png)

Can you see what’s wrong here? My second placeholder is 16! I tried to google around but I couldn’t find the exact problem here. If you know why, please welcome to leave a comment here. The second problem is that it didn’t give any warning for uninitialized variables.

After doing some searches, I finally managed to get it right by using a switch `-O`

Now, my output is as below:

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286807570/TVOrnKSka.png)](https://codecultivation.files.wordpress.com/2020/11/image-2.png)

The switch `-O` is basically a code optimization with compiler attempt to improve performance and/or code size at the expense of compilation time and possibly the ability to debug the program. If you are interested to find out more, please go to GCC website’s [Options That Control Optimization](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html#Optimize-Options).

Now, in order to help to detect uninitialized variables as warnings, I will need add one more switch which is `-Wall`.

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286808642/i_Syq3mfh.png)](https://codecultivation.files.wordpress.com/2020/11/image-3.png)

The switch `-Wall` helps to enable all warning flags including `-Wuninitialized` which warns if variable is used without first being initialized. More info about both switches can be found [here](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#index-Wall) and [here](https://gcc.gnu.org/onlinedocs/gcc/Warning-Options.html#index-Wuninitialized).

The last thing we need to do is to modify tasks.json to include both switches.

    {
         // See https://go.microsoft.com/fwlink/?LinkId=733558 
         // for the documentation about the tasks.json format
         "version": "2.0.0",
         "tasks": [
             {
                 "type": "shell",
                 "label": "gcc.exe build active file",
                 "command": "C:\mingw-w64\x86_64-8.1.0-posix-seh-rt_v6-rev0\mingw64\bin\gcc.exe",
                 "args": [
                     "-g",
                     "-O",
                     "-Wall",
                     "${file}",
                     "-o",
                     "${fileDirname}\${fileBasenameNoExtension}.exe"
                 ],
                 "options": {
                     "cwd": "C:\mingw-w64\x86_64-8.1.0-posix-seh-rt_v6-rev0\mingw64\bin"
                 },
                 "problemMatcher": [
                     "$gcc"
                 ],
                 "group": {
                     "kind": "build",
                     "isDefault": true
                 }
             }
         ]
     }

So when we run build in VS Code, it will automatically optimise codes and display warnings.

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286809930/T3qyeVMEI.png)](https://codecultivation.files.wordpress.com/2020/11/image-4.png)

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1662286811168/IUmlJZQMJ.png)](https://codecultivation.files.wordpress.com/2020/11/image-5.png)

Thanks for your time!
I recently had the misfortune of implementing parallel prefix sum on the GPU. It is a rather complicated algorithm, at least at first look, especially for someone who is rather new to GPU programming such as myself. I had to look through a couple articles and go over the algorithm in my head a few time before getting it working. Those articles I read were helpful and I wouldn't have gotten it working without them. They mostly focused on the theory and as a beginner I was more looking for a basic understanding of the algorithm and a low level look at its implementation. This is what I hope to give in this article.
## What is Prefix Sum?
Prefix sum on its own is rather simple. You have an array of numbers and you want the number at each index to be the sum of all the numbers in the previous indices.
![example input and output](/alexandre1357.github.io/assets/PrefixSum.png)
Doing this sequentially on the CPU is very easy. Simply loop through the array and keep track of the current sum. 
```c++
int array[10] = {...};
int sum = 0;

for (int i = 0; i < length(array); i++)
{
	int temp = sum;
	sum += array[i];
	array[i] = temp;
}
```

## Prior Reading
I'm going to be assuming that you know certain things while reading the rest of this article. I would include it all in the article but that would be a very long article and other people have already done a better job explaining them than I will. First thing I'm going to assume is that you are using opengl as that is what most people learn first for graphics/gpu programming or at least its what I learned first. So, if you know little to nothing about opengl you should go to [learnOpenGl](https://learnopengl.com/Getting-started/OpenGL) and go through at least the getting started section. Then if you don't know much about compute shaders or how to use them go to the [compute shader section](https://learnopengl.com/Guest-Articles/2022/Compute-Shaders/Introduction) on the same website. I am using Shader Storage Buffer Objects (SSBO) to modify data on the GPU which [opengl wiki](https://www.khronos.org/opengl/wiki/Shader_Storage_Buffer_Object) has a decent page on. If you are just looking for how to bind an SSBO it is at the bottom of that wiki page. When setting up the layout for the SSBO on the shader side a format for the buffer can be specified. There is std140 and std430 and you can find a general explanation on the [opengl wiki](https://www.khronos.org/opengl/wiki/Interface_Block_(GLSL)#Memory_layout) and the specific rules in the [opengl specs](https://registry.khronos.org/OpenGL/specs/gl/glspec45.core.pdf#page=159). Although all you need to know for this article is that std430 formats arrays of integers, and other primitive data types, the same way the array will be formatted by C++ which means you don't have to worry about the alignment of strides. Finally, I use some built-in compute shader variables all of which are listed on the [opengl wiki](https://www.khronos.org/opengl/wiki/Built-in_Variable_(GLSL)#Compute_shader_inputs).
## Parallel Prefix Sum
Now onto the hard part which is making the operation parallel. The algorithm commonly used to do this is derived from Blelloch's algorithm. I can't explain the mathematical theory of the algorithm as I don't fully understand it. There are sources below if you are interested in the theory. I can however visually explain the basic steps and pattern. The algorithm has two phases. The first phase is the up-sweep or reduce phase.
![Up Sweep](/alexandre1357.github.io/assets/UpSweep.png)
A threads job on any give step is to calculate the left and right index that it should be operating on. Then the right index gets assigned the sum of the left and right operands. Since each thread works on two indices the total number of threads needed to run the algorithm is equal to the number of elements halved. All of the threads are running at the start of this phase and then are reduced by half each step until only one thread runs on the last step. The stride, which is the distance between operands, is initialized to one and doubled each step. The left and right index can be calculated using the stride and index of the thread. The code for this is marked below with a comment. A couple other points are also marked which I mention in the paragraph below.
```glsl
#version 460 core

// work goup size <---
layout (local_size_x = 128, local_size_y = 1, local_size_z = 1) in;

layout (std430, binding = 0) buffer scan_buffer
{
	uint scan[];
};

void main()
{
	int threadIndex = int(gl_LocalInvocationID.x);
	
	int numElements = int(gl_WorkGroupSize.x) * 2;
	int stride = 1;
	
	for (int threads = numElements / 2; threads > 0; threads = threads / 2)
	{
// memory barrier <---
		memoryBarrierBuffer();
		barrier();

// cull unused threads <---
		if (threadIndex < threads)
		{
// calculation of left and right index <---
			int left_index = stride * (2 * threadIndex + 1) - 1;
			int right_index = stride * (2 * threadIndex + 2) - 1;

			scan[right_index] += scan[left_index];
		}

		stride *= 2;
	}
	
	...
```
This is the first part of the shader which includes the first phase of the algorithm. In this algorithm each step relies on the work done in the last step. So the function memoryBarrierBuffer is called to make sure any writes being executed finish before going to the next step. The barrier function is just after to make sure all threads reach this point before the next step as well. These functions only work on groups of invocations so the number of elements that can be sorted by the shader is equal to double the work group size since each thread works on two elements. I should note here that due to the pattern the algorithm follows the number of elements that can be sorted at once is 2^n. Once again there are resources below for a better explanation of the theory. Finally each step requires fewer threads so an 'if' statement simply checks if the local index is less then the number of active threads to determine if it should do the work. Now onto the second phase which is the down-sweep phase.
![Down Sweep](/alexandre1357.github.io/assets/DownSweep.png)
At this point in the algorithm the last element in the array is equal to the sum of the entire array. If you need that number for something in your program you should assign it to some other variable because the first step of this phase is to assign the last element in the array to zero. The pattern for the rest of this phase is essentially the reverse of the pattern from the reduce phase. The number of working threads is initialized to one this time and doubles each step. Whereas, the stride is initialized to the number of elements halved and is reduced by half each step. There is one more operation being done per thread. Each thread still assigns the right index to the sum of the left and right operands but this time the left index is assigned to the value that was in the right index before the operands were summed. That part is probably a bit clearer if you look at the arrows in the diagram above or the code below.

```glsl
	int threadIndex = int(gl_LocalInvocationID.x);
	
	int numElements = int(gl_WorkGroupSize.x) * 2;
	int stride = 1;
	
	...
	
// first step <---
	if (threadIndex == 0)
	{
		uint sumIndex = (numElements - 1);
		scan[sumIndex] = 0;
	}
	
	stride = numElements / 2;
	
	for (int threads = 1; threads < numElements; threads *= 2)
	{
// memory barrier <---
		memoryBarrierBuffer();
		barrier();
		
		if (threadIndex < threads)
		{
// calculation of left and right index <---
			int left_index = stride * (2 * threadIndex + 1) - 1;
			int right_index = stride * (2 * threadIndex + 2) - 1;
			
// move and sum <---
			uint temp = scan[right_index];
			scan[right_index] += scan[left_index];
			scan[left_index] = temp;
		}
		
		stride = stride / 2;
	}
}
```
This is the last section of the algorithm. It looks fairly similar to the first. The memory barrier is still there to make sure the memory and threads are ready for each step. Also the calculation for the left and right index are the same. There is some extra code at the start to set the last element to zero and the calculations for stride and the number of threads are swapped. Finally there is a move as well as a sum being done on the left and right indices. And that's it for parallel prefix sum. However, there is a limit on the number of elements that this can sort. As mentioned earlier, each step relies on the last. Threads can't sync up with threads outside their group which means the number of elements the shader can sort is equal to double the group size. Now the group size can be increased but there is a limit on how big a group can get. Also, that means the shader is locked to only sorting that many elements. Fortunately, there is a better way.
## Group Sum
The solution, more parallel prefix sum. If you understand and implemented the above this step is fairly easy. How this works is that we have another array which keeps track of the sum of each group of elements. We then run prefix sum on the array of group sums which gives us how much each group is off by and add that amount to the elements in the original array.
![Group Sum](/alexandre1357.github.io/assets/GroupSum.png)

Just as the size of each group of elements has to be 2^n the group sum array also has to have a length of 2^n since it is also being run through parallel prefix sum. Before I get to the shader for the group sum the original prefix sum shader has to be updated. 
```glsl
#version 460 core

layout (local_size_x = 128, local_size_y = 1, local_size_z = 1) in;

layout (std430, binding = 0) buffer scan_buffer
{
	uint scan[];
};

// new buffer <---
layout (std430, binding = 1) buffer group_sum_buffer
{
	uint groupSum[];
};

void main()
{
// new indices to keep track of <---
	int localThreadIndex = int(gl_LocalInvocationID.x);
	int globalThreadIndex = int(gl_GlobalInvocationID.x);
	int groupIndex = int(gl_WorkGroupID);
	
	int numElements = int(gl_WorkGroupSize.x) * 2;
	int stride = 1;
	
	for (int threads = numElements / 2; threads > 0; threads = threads / 2)
	{
		memoryBarrierBuffer();
		barrier();

// local thread index still used to cull threads <---
		if (localThreadIndex < threads)
		{
// using global thread index instead of local index <---
			int left_index = stride * (2 * globalThreadIndex + 1) - 1;
			int right_index = stride * (2 * globalThreadIndex + 2) - 1;

			scan[right_index] += scan[left_index];
		}

		stride *= 2;
	}

	if (localThreadIndex == 0)
	{
		uint sumIndex = (numElements - 1);
// assign the group sum <---
		groupSum[groupIndex] = scan[sumIndex];
		scan[sumIndex] = 0;
	}
	
	stride = numElements / 2;
	
	for (int threads = 1; threads < numElements; threads *= 2)
	{
		memoryBarrierBuffer();
		barrier();
		
		if (localThreadIndex < threads)
		{
			int left_index = stride * (2 * globalThreadIndex + 1) - 1;
			int right_index = stride * (2 * globalThreadIndex + 2) - 1;
			
			uint temp = scan[right_index];
			scan[right_index] += scan[left_index];
			scan[left_index] = temp;
		}
		
		stride = stride / 2;
	}
}
```
First thing to mention is the new buffer at the top to hold the group sums. As mentioned before, in between the first and second phase, the last element in the scan array is equal to the sum of the whole array. So, at that point the group sum is assigned to the array at the same index as the work group index. Last thing to note is that instead of using the local thread index to calculate the left and right indices the global thread index is used. The local index is still used in the 'if' statement to cull threads that don't need to do work on that step. Now that this is updated we can move onto the prefix sum for the group sum.
```glsl
#version 460 core

layout (local_size_x = 128, local_size_y = 1, local_size_z = 1) in;

layout (std430, binding = 1) buffer group_sum_buffer
{
	uint groupSum[];
};

void main()
{
	int threadIndex = int(gl_LocalInvocationID.x);
	
	int numElements = int(gl_WorkGroupSize.x) * 2;
	int stride = 1;
	
	for (int threads = numElements / 2; threads > 0; threads = threads / 2)
	{
		memoryBarrierBuffer();
		barrier();

		if (threadIndex < threads)
		{
			int left_index = stride * (2 * threadIndex + 1) - 1;
			int right_index = stride * (2 * threadIndex + 2) - 1;

			groupSum[right_index] += groupSum[left_index];
		}

		stride *= 2;
	}

	if (threadIndex == 0)
	{
		uint sumIndex = (numElements - 1);
		groupSum[sumIndex] = 0;
	}
	
	stride = numElements / 2;
	
	for (int threads = 1; threads < numElements; threads *= 2)
	{
		memoryBarrierBuffer();
		barrier();
		
		if (threadIndex < threads)
		{
			int left_index = stride * (2 * threadIndex + 1) - 1;
			int right_index = stride * (2 * threadIndex + 2) - 1;
			
			uint temp = groupSum[right_index];
			groupSum[right_index] += groupSum[left_index];
			groupSum[left_index] = temp;
		}
		
		stride = stride / 2;
	}
}
```
This is an exact copy of prefix sum I showed earlier in the article. The only difference is the name and binding of the buffer. The last thing to do is to add the group sums back to the original array. You should be able to write that shader on your own as it is quite simple compared to the rest of the algorithm. 

## Applications of Prefix Sum
I implemented this algorithm for a project that I was working on as a second year programming student at BUas. The project was to make a GPU accelerated particle system. All the memory for the particles had to be on the GPU and because of how shaders and rendering work the memory had to be contiguous. If I wanted to kill one or more particles and remove them from the list, I couldn't do something simple like replacing them with live particles from the back of the list. So, I had to implement an algorithm called vote scan compact. First it votes on which particles are alive with a one and which are dead with a zero. That data is copied over to the scan buffer and prefix sum is ran on the buffer. The resulting list contains the new indices for each particle. For each index the particle is copied to its new index in the scan array if it was marked alive in the vote buffer.
![Vote Scan Compact](/alexandre1357.github.io/assets/VoteScanCompact.png)
Part of the reason I said "misfortunate" at the beginning of this article is because there turned out to be a simpler solution in my situation. The particles in my system have a life time that is randomly picked in a range and nothing else determines when they die. This means that I can just estimate how many were left alive and only update and render that many particles. 
## Conclusion 
Parallel prefix sum can be a difficult algorithm to implement. If you are somewhat new to GPU programming it is even harder. Although parallel prefix sum is useful and needed in certain situations it is also complex. If you can think of a simpler solution that works use that instead. This article was only a look at the basics. There is more to be done to optimize the algorithm and make it work on an even larger number of elements. But I hope this article helped you understand how to implement the basic version. If it didn't or you want more there are some resources down below. Thanks for reading.


## Resources
Here are some articles that I used to research parallel prefix sum. They also include some advanced optimizations that I didn't go over. There are plenty of more articles on the topic if you simply search 'parallel prefix sum'.
- [https://developer.nvidia.com/gpugems/gpugems3/part-vi-gpu-computing/chapter-39-parallel-prefix-sum-scan-cuda](https://developer.nvidia.com/gpugems/gpugems3/part-vi-gpu-computing/chapter-39-parallel-prefix-sum-scan-cuda)
- [https://cs.wmich.edu/gupta/teaching/cs5260/5260Sp15web/lectureNotes/thm14%20-%20parallel%20prefix%20from%20Ottman.pdf](https://cs.wmich.edu/gupta/teaching/cs5260/5260Sp15web/lectureNotes/thm14%20-%20parallel%20prefix%20from%20Ottman.pdf)

Paper by Guy Blelloch on parallel prefix sum.
- [https://www.cs.cmu.edu/~guyb/papers/Ble93.pdf](https://www.cs.cmu.edu/~guyb/papers/Ble93.pdf)

This video is where I first heard about parallel prefix sum. It doesn't include an explanation of the algorithm or how to implement it but it does give another use case and a link to source code with the implemented algorithm.
- [https://www.youtube.com/watch?v=jw00MbIJcrk&t=430s](https://www.youtube.com/watch?v=jw00MbIJcrk&t=430s)

&emsp;

![BUas logo](/alexandre1357.github.io/assets/BUas-Logo.png)

&emsp;
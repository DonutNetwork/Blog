This is a little dev blog about how we reduced our overworld disk usage by 2x and improved our CPU usage/ram usage alongside reducing load on the disk.

Alright to start at first we have our basic Region format from Minecraft, pretty much an extremely old format that has lots of reasons, I assume that one of them was to not have one-file per chunk (This is a File format "issue" in systems with limitations)
due to file finding and listings, we had to make changes on our file systems to increase the number of files we could hold on directories.

To start off with the Region format, one-file contains 1024 chunks, the way it's stored is explained on the minecraft guides, basically a lot of disk usage due to how it stores and knows in which sections of bytes the chunk data is.
New format created by Xymb I believe, Linera format. This format loads the entire file data into memory, this keeps a "basic" concept of the region format, with the difference that it does not have "knowledge on which section of the file the chunks are at, including some less garbage such as longs for timelines/
The problem with the new format is: High ram usage, because it keeps the basics of teh region format, this operation is still >> 5, meaning the chunk files are still 1024 chunks per file, effectively speaking by loading one file you're ffectively loading 1024 chunks data into memory. Which can be intensive, when you consider
the fact that the RegionFileStorage can hold up to n region files (configured by the user but by default 128 files iirc). This can be: slow loading, due to reading a "huge" file, you can endup with "more" cpu usage and memory usage, while saving the disk.

How we managed to solve the problem:\

The first step is, we implemented an actual double-layer compression to reduce the memory usage even more while cached, Linear uses LZ4 compression which is def a good side, but if applying a 2 layer we observed a "low" cost for huge benefit in memory, with up to 25% less heap memory usage,
the second change was to write a custom converter, because of the fact that linear uses 1024 I didn't think it was a good approach for it, due to the fact that a player can load up to 4 different >> 5, meaning he can effectively load 4096 chunks into memory with the linear format, which isn't good,
so the next change is the reason why we had to enable big directories in linux, the section usage changed from >> 5, we no longer have 1024 chunks per file, instead we have 16, and the difference in performance here is actually big, we understand that this is hardware related as well, so for our systems we run NVMEs,
and the latency is extremely small, giving us a sub 1ms open per file, which is extremely fast considering that on Linear the usage could go from 30 to more than 100ms per file.
\
This allowed us to have an extremely low memory usage while having every single benefit from the so called Linear format, our overall benefits from our overworld was a 2x size reduction, going from 5TB of usage to 2.5TB, and from our end dimension going from 2TB to just over 170GB (due to empty chunks),
including less CPU usage as well overall.
\
Now onto some other small cahnce on the region storage, currently we do not have the "n limit", I believe the N limit is good for servers but it's not as effective, instead of that we keep a 60 second deadline old file cache with counters, meanwhile there's chunks loaded onto it, the file will remeain
in memory, and if the file unloads, effectively decreasing the counter. once the counter is at 0, the deadline will close and write to the file.

# Lab1 Checkpoint 1

基本上就这仨要求

> In principle, then, the Reassembler will have to handle three categories of knowledge: 
>
> 1. Bytes that are the next bytes in the stream. The Reassembler should push these to the stream (output .writer()) as soon as they are known. 
> 2. Bytes that fit within the stream’s available capacity but can’t yet be written, because earlier bytes remain unknown. These should be stored internally in the Reassembler.
> 3. Bytes that lie beyond the stream’s available capacity. These should be discarded. The Reassembler’s will not store any bytes that can’t be pushed to the ByteStream either immediately, or as soon as earlier bytes become known.



![image-20241009114454352](https://cdn.jsdelivr.net/gh/NGSJCBF/img@main/img/202410091144406.png)




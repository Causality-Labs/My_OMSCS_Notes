
**• What are the advantages and disadvantages of a layered architecture?**
- Advantages
	- Scalability
	- Modularity
	- Flexibility to add or delete components
		- makes for cost effective implementations
	- Each layer uses service from layer below and provides a service to layer above
- Disadvantages
	- Some layers' functionality depends on the information from other layers, which can violate the goal of layer separation.
	- One layer may duplicate lower-layer functionalities. For example, the functionality of error recovery can occur in lower layers but also in upper layers as well
	- Additional overhead that is caused by the abstraction between layers
• **What are the differences and similarities between the OSI model and the five-layered Internet**
**model?**
- Five layered Internet Protocol Stack looks as follows:
	- Application layer 
	- Transport layer 
	- Network layer 
	- Data Link layer 
	- Physical layer
- OSI Model (Seven Layer Open Systems Interconnection Stack)
	- Application layer 
	- Presentation layer
	- Session Layer
	- Transport layer 
	- Network layer 
	- Data Link layer 
	- Physical layer
- Similarities:
	- The Transport, Network, Data Link and Physical layer are similar
- Differences
	- OSI is 7 layers and Internet Stack is 5 layes because the Session and  Presentation layers are combined with the Application Layer in the internet protocol stack.
- The interface between the Application and Transport layer is called sockets.

• **What are sockets?**
- The interface between the Application and Transport layer is called sockets

• **Describe each layer of the OSI model.**
	- Application layer
		- Includes Multiple Protocols:
			- HTTP (web)
			- SMTP (e-mail)
			- FTP (transfers files between two end hosts)
			- DNS (Translates domains names to IP addresses)
		- Offers multiple services, interfaces and protocols and interfaces depending on application
		- At this layer a packet of information is called a message
	- Presentation layer
		- Intermediate role of formatting the information that it receives from the layer below and delivering it to the application layer.
		-  Some functionalities of this layer are formatting a video stream or translating integers from big endian to little endian format.
	- Session Layer
		- Manages different transport streams belonging to the same session and end-user application process.
		- For example, in the case of a teleconference application, it is responsible to tie together the audio stream and the video stream.
	- Transport layer 
		- Responsible for end to end communication between end hosts
		- Two transport protocols
			- Transmission Control Protocol (TCP)
				- Certified Letter, reliable and managed.
				-  **Connection-Oriented:** Establishes a formal connection before sending data.
				- **Guaranteed Delivery:** Confirms the message arrived; if not, it resends it.
				- **Flow Control:** Ensures the sender doesn't overwhelm the receiver (Speed Matching).
				- **Congestion Control:** Slows down transmission if the network itself is "clogged."
			- User Datagram Protocol (UDP)
				- Regular mail, fast and simple
				- **Connectionless**: Sends data without checking if the receiver is ready
				- **Best Effort**: No garuntee is data arrives, no retransmissions.
				- **No Controls**: Lack flow or congestion management. faster but less polite to the network
			- Unit of data in this layer is called a segment.
	- Network layer
		- Packet of information is called a datagram in the Network Layer.
		- 
	- Data Link layer 
	- Physical layer

• Provide examples of popular protocols at each layer of the five-layered Internet model.
• What is encapsulation, and how is it used in a layered model?
• What is the end-to-end (e2e) principle?
• What are the examples of a violation of e2e principle?
• What is the EvoArch model?
• Explain a round in the EvoArch model.
• What are the ramifications of the hourglass shape of the internet?
• Repeaters, hubs, bridges, and routers operate on which layers?
• What is a bridge, and how does it “learn”?
• What is a distributed algorithm?
• Explain the Spanning Tree Algorithm.
• What is the purpose of the Spanning Tree Algorithm?

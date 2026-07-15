Hi Shuxue,
Thanks for your inputs.
We agree that performance optimization on the backend should be explored wherever possible. However, our concern is not limited to the current volume (100–200 payees or 90 days of transactions), but also to scalability as the data grows over time.
A few points for consideration:
Returning the complete dataset for every request may work for smaller volumes today, but it does not address the long-term scalability requirements.
The performance impact is not only at the database/Redis layer. Serialization/deserialization, network transfer, API processing time, and frontend rendering also contribute to the overall latency.
Different markets may have different transaction volumes and usage patterns. Therefore, the current implementation in other markets may not necessarily be suitable for VN requirements.
Pagination is generally the recommended approach for handling growing datasets as it provides predictable performance and improves the user experience.
That said, as an interim solution, we can certainly evaluate backend optimizations for handling the current data volume (e.g., query optimization, caching improvements, and response optimizations). If the expected volume remains within acceptable limits, this may help minimize immediate frontend changes.
We would recommend aligning on the expected data growth and volume requirements before deciding whether an optimization-only approach is sufficient or if pagination support would be required as a scalable solution.
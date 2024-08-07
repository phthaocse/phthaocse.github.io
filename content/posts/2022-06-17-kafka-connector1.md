---
title: Xây dựng data pipeline với Kafka connector 
date: 2022-06-17T17:35:00+07:00
layout: post
summary: Trong các hệ thống hiện đại ngày nay, khi dữ liệu và dữ liệu là vô hạn thì việc xây dựng hệ thống truyền tải và chuyển hoá dữ liệu đóng vai trò rất quan trọng.
tags:  ["IT", "Kafka"]
categories: [Technology]
---
# Xây dựng data pipeline với Kafka connector - Phần 1: Data Pipeline

Hôm nay chúng ta sẽ tìm hiểu về Kafka connector và cách Kafka connector được dùng để xây dựng data pipeline trong các hệ thống phần mềm hiện đại

## Data pipeline và những điều cần biết

Xây dựng data pipeline là một chủ đề và lĩnh vực khá rộng, nên trong phần này chúng ta chỉ tìm hiểu những khía cạnh quan trọng và có liên quan đến quyết định sử dụng Kafka connector

### Timeliness

- Trong những hệ thống hiện đại ngày nay nhu cầu về việc lưu trữ và sử dụng dữ liệu là rất đa dạng. Một hệ thống vừa phải đáp ứng được việc data cần phải realtime như các hệ thống dashboard để người sử dụng có thể có cái nhìn và monitor những hoạt động liên quan một cách chính xách. Nhưng cũng với những data như vậy, có những hệ thống chỉ cần truy cập và sử dụng nó mỗi ngày hay mỗi tháng một lần. Đó chính là tính thời điểm (timelines), một trong những yếu tố quan trọng mà ta cần phải quan tâm, hầu hết các hệ thống được xây dựng để phù hợp dựa trên những yêu cầu như vậy. Và một hệ thống tốt được xem là có hỗ trợ tốt hầu hết cho những requirements mang tính thời điểm khác nhau, đồng thời có thể migrate data dễ dàng giữa các mô hình data base trên thời điểm một khi các business reuirement thay đổi. 

- Những yêu cầu trên đều có thể được hỗ trợ bởi Kafka, Producer trong Kafka có thể thực hiện việc ghi data ở một tần suất liên tục (sử dụng cho các hệ thống data real time) hoặc ghi data một lần nhưng với một khối lượng lớn data (các hệ thống phân tích big data), và điều này cũng tương tự với Consumer. Nhưng Producer và Consumer lại là hai thành phần tách rời lẫn nhau, nên vì thế chúng có thể nhận các nhiệm vụ ở các role khác nhau như việc Producer có thể ghi data một cách liên tục trong khi Consumer thì đọc data the từng sự kiện hoặc thời khoá biểu định sẵn và ngược lại.

### Reliability

- Đây là một yếu tố quan trọng và không thể thiếu trong tất cả các hệ thống máy tính. Một hệ thống muốn chạy tốt và phục vụ được trong môi trường production luôn phải đảm bảo rằng nó không có single points of failure cũng như có thể nhanh chóng phục hồi (resilient) trước bất kỳ failure nào. Điều này càng quan trọng trong những hệ thống liên quan đến data, data là tất cả của bất kỳ doanh nghiệp hiện đại nào vì thế việc đảm bảo data được bảo toàn là một bài toán quan trọng hàng đầu khi xây dựng data pipeline. Kafka cung cấp cho engineers những giải pháp đảm bảo toàn vẹn dữ liệu, vì đây là một topic cũng tương đối nhiều kiến thức nên chúng ta sẽ cùng tìm hiểu trong một post khác.

### High and Varying Throughput

- Một vấn đề khác luôn đực cân nhắc đầu tiên trong các enterprise system đó là tính scalable, bất kỳ một hệ thống enterprise nào về lâu dài đều sẽ scale theo những cách mà ta khó kiểm soát được vì thế những hệ thống như vậy luôn phải được thiết kế để scale và scale tốt, khi throughput tăng đột ngột nó vẫn có thể xử lý được. Kiến trúc kafka cho phép việc kafka như là một buffer giữa Consumer và Producer, một khi throughput của Producer tăng đột ngột khiến Consumer không thể tiêu thụ kịp thì Kafka sẽ tích luỹ phần data đó đến khi nào Consumer có thể handle. Ngoài ra Kafka còn cho phép chúng ta chủ động scale cách thêm vào nhiều consumers hoặc producers một cách độc lập để phù hợp với nhu cầu. Nhưng bất cứ gì đều có tradeoff, khi việc scale up đồng nghĩa với việc resource tăng nên cần phải kiểm soát resource thật tốt và Kafka cũng cung cấp một số giải pháp để kiểm soát điều này mà trong bài viết nay không đề cập đến

### Data Format

- Data format là một yếu tố quan trọng hàng đầu khác, việc chuyển đổi data qua các hệ thống lưu trữ khác nhau với từng cấu trúc và format dữ liệu khác nhau đòi hỏi một bộ chuyển đổi cho các loại data cụ thể. Chẳng hạn như việc chúng ta cần load data từ relationship database vào Elasticsearch để phục vụ việc search hoặc từ SQL cần sync data sang một NoSQL phục vụ cho các business requirements. Xây dựng những bộ chuyển đổi như vậy không phải là một công việc đơn giản, đặc biệt nếu hệ thống dữ liệu sử dụng nhiều loại data storage. Kafka connector được sinh ra cũng vì mục tiêu tinh giảm bớt công việc chuyển đổi như vậy, có thể xem Kafka connector như một adapter, nơi mà chúng ta có thể gắn vào bộ chuyển đổi mà ta mong muốn để có thể dễ dàng migrate. Điều đó cho thấy Kafka Connect là một agnostic engine không phụ thuộc vào bất kỳ data store nào.

### Transformations

- Có hai phương pháp chính để transform data đó là ETL và ELT
- ETL 
    - Là viết tắt của Extract-Transform-Load, data pipeline sẽ đóng vai trò trong việc xử lý và chính sửa data theo business requirement trước khi nó được chuyển tới target storage để sử dụng.
    - Lợi ích của mô hình này nằm ở việc giúp tiết kiệm được thời gian xử lý và bộ nhớ, chúng ta sẽ không cần phải lưu trữ dữ liệu thô sau đó xử lý và lưu dữ liệu mới tại target storage, vì tất cả bước xử lý đã do pipeline đảm nhận 
    - Nhưng việc xử lý dữ liệu trong pipeline đôi khi dẫn đến việc quá tải của pipeline trong việc tính toán và xử lý. Ngoài ra trong trường hợp dữ liệu cần được xử lý với nhiều mục đích xa hơn so với yêu cầu ban đầu, thì đây là điểm yếu của phương pháp này vì chúng ta không thể tận dụng lại dữ liệu đã được xử lý bởi pipeline.

- ELT:
    - So với ETL chúng ta sẽ đảo hai step cuối để ra được phương pháp thứ hai là Extract Load Transform. Nghe tới đây chắc chúng ta cũng thấy được khác biệt cơ bản chính là việc data sẽ được load lên trước khi được xử lý đồng nghĩa với việc pipeline chỉ chịu trách nhiệm vận chuyện lượng data thô ban đầu đến target storage.
    - Vì là hai phương pháp có thể coi là nghịch đảo của nhau nên ta thấy lợi ích của phương pháp này là nhược điểm của phương pháp còn lại, trong trường hợp của ELT thì lợi ích chính đến từ việc data raw nên có thể đáp ứng mọi nhu cầu xử lý về sau mà không cần phải load lại dữ liệu mới. Còn nhược điểm thì như ta cũng đã biết, hệ thống nguồn sẽ tốn rất nhiều resource và lưu trữ cho việc xử lý data.    

### Security 

- Vấn đề về bảo mật luôn được quan tâm ở bất cứ máy tính nào và đặc biệt quan trọng với dữ liệu. Các vấn đề liên quan tới bảo mật mà chúng ta cần xem xét cho hệ thống
    - Ai là người có quyền truy cập vào data đã được đưa vào trong Kafka xử lý ?
    - Data trong pipeline trên toàn bộ quy trình đã được mã hoá hay chưa ?
    - Những ai được can thiệp để thay đổi pipeline ?
    - Trong trường hợp dữ liệu được đọc và ghi từ những nơi đòi hỏi authen, thì việc authen đó đã hoạt động đúng ?
    - Dữ liệu liên quan đến định danh cá nhân có được lưu và sử dụng một cách hợp phjáp hay không ?

- Để trả lời cho những vấn đề trên, Kafka cung cấp các giải pháp liên quan đến việc encrypt data trong hệ thống pipeline, audit log để track các hoạt động truy cập. Ngoài ra Kafka Connect cũng cấp các chớ authen với external System. Chi tiết hơn về vấn đề security của Kafka chúng ta sẽ bàn trong những bài viết khác.

## Coupling and Agility

*Ad hoc pipeline*

Như cái tên đã nói lên, đó chính là việc xây dựng pipeline dựa trên nhu cầu về business tại thời điểm hiện tại, gắn chất chúng với nhau như việc source storage và target storage phải biết và phụ thuộc lẫn nhau. Ví dụ khi cần chuyển đổi data từ MySQL sang Oracle, chúng ta chỉ xây dựng pipeline riêng biệt cho hai hệ cơ sở dữ liệu này. Điều này dẫn đến khi scale hoặc chính yêu cầu liên quan đến business cần sử dụng một db khác trong tương lai thì cost cho việc refactor cho toàn bộ hệ thống là rất lớn vì chúng ta không thể tận dụng lại hệ thống cũ, và khi xây dựng hệ thống mới việc migrate data cũng là vấn đề hết sức đau đầu.

*Loss of metadata*

- Metadata ở đây trong trường hợp này chính là schema, khi không có sự có mặt của schema, bất kỳ một thay đổi liên quan đến data structure của một data point trong toàn hệ thống sẽ dẫn đến break toàn bộ các hệ thống khác nếu không có sự nhận biết về thay đổi đó. Trong trường hợp như vậy tất cả các điểm khác trong hệ thống cần thông tin về việc data thay đổi để tiến hành việc hiện thực chuyển đổi data hợp lý. Với việc support schema evolution trong pipeline, Kafka giúp cho các application có thể thay đổi mà không sợ việc làm break các hệ thống khác.

*Extreme processing* 

Các downstream system sẽ cần data dựa trên những yêu cầu về business khác nhau dẫn đến yêu cầu về data từ pipeline thay đổi liên tục do việc xử lý data được handle bởi pipeline. Điều này làm cho pipeline thay đổi liên tục theo sự thay đổi của hệ thống downstream dẫn đến việc không hiệu quả, an toàn, và linh hoạt. Vì thế để đảm bảo tính linh hoạt của pipeline chúng ta luôn đảm bảo data "raw" nhất có thể.

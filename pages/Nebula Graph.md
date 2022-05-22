type:: [[Database]]
language:: [[CPP]]
category:: [[Graph]]

- Nebula Graph 是一个开源的图数据库，
-
- 图数据库使用点和边来表达数据之间的关系，主要的场景包括社交网络，知识图谱等。
-
- 用起来的感觉大概是这样(以[图数据库体操](https://siwei.io/resolve-wordle/#%E5%9B%BE%E8%B0%B1%E5%85%A5%E5)为例)：
	- 关系建模
		- ```sql
		  
		  CREATE SPACE IF NOT EXISTS chinese_idiom(partition_num=5, replica_factor=1, vid_type=FIXED_STRING(24));
		  USE chinese_idiom;
		  # 创建点的类型
		  CREATE TAG idiom(pinyin string); #成语
		  CREATE TAG character(); #汉字
		  CREATE TAG character_pinyin(tone int); #单字的拼音
		  CREATE TAG pinyin_part(part_type string); #拼音的声部
		  # 创建边的类型
		  CREATE EDGE with_character(position int); #包含汉字
		  CREATE EDGE with_pinyin(position int); #读作
		  CREATE EDGE with_pinyin_part(part_type string); #包含声部
		  ```
	- 匹配结果
		- ```sql
		  # 匹配成语中的一个结果
		  MATCH (x:idiom) "爱憎分明" RETURN x LIMIT 1
		  
		  # 返回结果
		  ("爱憎分明" :idiom{pinyin: "['ai4', 'zeng1', 'fen1', 'ming2']"})
		  ```
	- 按照音调进一步匹配
		- ```sql
		  # 有一个非第一个位置的字，拼音是 4 声，韵母是 ai，但不是爱
		  MATCH (char0:character)<-[with_char_0:with_character]-(x:idiom)-[with_pinyin_0:with_pinyin]->(pinyin_0:character_pinyin)-[:with_pinyin_part]->(final_part_0:pinyin_part{part_type: "final"})
		  WHERE id(final_part_0) == "ai" AND pinyin_0.character_pinyin.tone == 4 AND with_pinyin_0.position != 0 AND with_char_0.position != 0 AND id(char0) != "爱"
		  # 有一个一声的字，不在第二个位置
		  MATCH (x:idiom) -[with_pinyin_1:with_pinyin]->(pinyin_1:character_pinyin)
		  WHERE pinyin_1.character_pinyin.tone == 1 AND with_pinyin_1.position != 1
		  # 有一个字韵母是 ing，不在第四个位置
		  MATCH (x:idiom) -[with_pinyin_2:with_pinyin]->(:character_pinyin)-[:with_pinyin_part]->(final_part_2:pinyin_part{part_type: "final"})
		  WHERE id(final_part_2) == "ing" AND with_pinyin_2.position != 3
		  # 第四个字是二声
		  MATCH (x:idiom) -[with_pinyin_3:with_pinyin]->(pinyin_3:character_pinyin)
		  WHERE pinyin_3.character_pinyin.tone == 2 AND with_pinyin_3.position == 3
		  
		  RETURN x, count(x) as c ORDER BY c DESC
		  ```
-
- 第一感觉是比 SQL 还要复杂不少，需要理解点和边的概念
-
- 参考资料
	- [图数据库体操：用 Nebula Graph 搭成语图谱解汉兜](https://siwei.io/resolve-wordle/)
	- [图数据库的应用场景有哪些？](https://www.zhihu.com/question/58988651)
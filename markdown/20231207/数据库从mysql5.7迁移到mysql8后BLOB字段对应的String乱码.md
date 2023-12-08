# 数据库从mysql5.7迁移到mysql8后BLOB字段对应的String乱码

之前数据库使用的是mysql5.7，文章内容和评论内容我们使用了mysql的BLOB类型对应java的String类型，在之前的使用过程中没有显示出任何的问题。

但是后面我打算迁移到mysql8，因为不想有两个数据库对于我这么个小项目来说，nacos的数据库和应用的数据库公用一个服务就行了，把服务器上原来的mysql服务停用或者做备份。

本来以为很简单不会有任何问题，而且说mysql8会向下兼容mysql5.7，结果同步完数据库之后启动服务发现文章内容和评论内容里的汉字都乱码了。

但是我用navicat从数据库查询的时候发现中文显示是正常的

```
select cast(commentContent as CHAR) from `comment` where id=78;
select convert(commentContent USING utf8mb4) from `comment` where id=78;
```

因为是乱码的缘故，因此我考虑是不是数据库或者表或者字段的编码的问题，发现确实有问题，原来有些表是mysql5.7的utf8，到了mysql8是utf8mb3，其实是一样的，正常应该使用utf8mb4

于是我把原库导出sql，修改sql中的utf8改为utf8mb4，utf8_general_ci改为utf8mb4_general_ci

并且将数据库删除重建确认都是utf8mb4，但是结果还是一样。

实际上关于utf8，utf8mb3和utf8mb4的定义可以知道，utf8mb3是最大占用3字节，而utf8mb4是最大占用4字节，都是可变长度，顶多utf8mb4多了一些罕见字和其他字符，并不会产生全文中文汉字乱码的问题，而且原来的utf8和utf8mb3其实是保持一致的。

于是我考虑是不是mysql8的BLOB字段与java的字段映射是不是和mysql5.7的不太一样。

经过一番查询发现，mysql的BLOB对应的是java的byte[]类型，java和mysql的类型转换由ORM框架即mybatis的typeHandler处理，默认的BLOB对应的就是

```
typeHandler="org.apache.ibatis.type.BlobTypeHandler"
```

这个只能转换成byte[] bytes，其实自己可以转换，默认就是，不需要转

如果要修改这个转换类型，需要自定义typeHandler

参考：[MyBatis-Plus+自定义Hanlder解决Blob类型查询及插入问题](https://blog.csdn.net/itgirl2580/article/details/129377176)

```
package abc.com.blog.model.handler;

import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.MappedJdbcTypes;
import org.apache.ibatis.type.MappedTypes;
import org.springframework.stereotype.Component;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.sql.*;

/**
 * @description: mysql8的BLOB类型转换为String(mysql5.7的时候似乎可以自己转换，但是换到mysql8就不行了)
 * @date: 2023/12/07 下午5:32
 * @version: 1.0
 **/
@MappedJdbcTypes(JdbcType.BLOB)
@MappedTypes(String.class)
@Component
public class CustomString2BlobTypeHandler extends BaseTypeHandler<String> {

    @Override
    public void setNonNullParameter(PreparedStatement preparedStatement, int i, String parameter, JdbcType jdbcType) throws SQLException {
        // 声明一个输入流对象
        ByteArrayInputStream bis = null;
        long size = 0;
        try {
            byte[] bytes = parameter.getBytes("utf-8");
            // 把字符串转为字节流
            size = bytes.length;
            bis = new ByteArrayInputStream(bytes);
        } catch (Exception e) {
            throw new RuntimeException("Blob Encoding Error!");
        } finally {
            if (bis != null) {
                try {
                    bis.close();
                } catch (IOException e) {
                    throw new RuntimeException("Blob Encoding Error!");
                }
            }
        }
        //注意这里size是bytes.length()而非parameter.length()，网上一堆抄错的
        preparedStatement.setBinaryStream(i, bis, size);
    }

    @Override
    public String getNullableResult(ResultSet resultSet, String columnName) throws SQLException {
        Blob blob = resultSet.getBlob(columnName);
        String returnValue = null;
        if (null != blob) {
            returnValue = new String(blob.getBytes(1L, (int) blob.length()));
        }
        return returnValue;
    }

    @Override
    public String getNullableResult(ResultSet resultSet, int columnIndex) throws SQLException {
        Blob blob = resultSet.getBlob(columnIndex);
        String returnValue = null;
        if (null != blob) {
            returnValue = new String(blob.getBytes(1L, (int) blob.length()));
        }
        return returnValue;
    }

    @Override
    public String getNullableResult(CallableStatement callableStatement,  int columnIndex) throws SQLException {
        Blob blob = callableStatement.getBlob(columnIndex);
        String returnValue = null;
        if (null != blob) {
            returnValue = new String(blob.getBytes(1L, (int) blob.length()));
        }
        return returnValue;
    }
}

```

xml：

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="abc.com.blog.mapper.CommentMapper">

    <resultMap id="BaseResultMap" type="abc.com.blog.model.po.Comment">
        <id property="id" column="id" jdbcType="BIGINT"/>
        //省略
        <result property="commentLevel" column="commentLevel" jdbcType="VARCHAR"/>
        <result property="commentContent" column="commentContent" typeHandler="abc.com.blog.model.handler.CustomString2BlobTypeHandler"/>
        <result property="commentTime" column="commentTime" jdbcType="TIMESTAMP"/>
        <result property="deleted" column="deleted" jdbcType="INTEGER"/>
    </resultMap>
</mapper>
```

但是上面的设置完之后发现还是不行，因为上面针对的是xml的配置

如果我们使用了mybatis-plus的QueryWrapper,则需要在实体类PO对应的类和字段上加上注解


实体类上打上注解：@TableName(value = "数据库表名", autoResultMap = true)

实体类对应的字段上加上注解

```
 @TableField(typeHandler = IntegerListTypeHandler.class)
 private List<Integer> copyTo;
```

参考：[在Mybatis-Plus中指定TypeHandler后不生效的问题与解决办法](https://blog.csdn.net/qq_33238562/article/details/129181964)

PO:

```
package abc.com.blog.model.po;

import abc.com.blog.model.enumeration.CarrierTypeEnum;
import abc.com.blog.model.enumeration.CommentLevel;
import abc.com.blog.model.handler.CustomString2BlobTypeHandler;
import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.annotation.TableField;
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.fasterxml.jackson.annotation.JsonFormat;
import io.swagger.annotations.ApiModelProperty;
import lombok.Data;

import java.time.LocalDateTime;

/**
 * @description: 评论类
 * @date: 2023/09/19 0:48
 * @version: 1.0
 **/
@Data
@TableName(value = "comment",autoResultMap = true)
public class Comment {
    @TableId(type = IdType.AUTO)
    private Long id;


    @ApiModelProperty(value = "评论级别")
    private CommentLevel commentLevel;

    @ApiModelProperty(value = "评论内容")
    @TableField(typeHandler = CustomString2BlobTypeHandler.class)
    private String commentContent;

    @ApiModelProperty(value = "评论时间")
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime commentTime;

    @ApiModelProperty(value = "是否已删除")
    private Boolean deleted;
}

```

---

除此之外，我看到的更多的办法是实体类中多加一个byte[]类型定义@TableField，然后让orm对应到这个byte[]类型字段，同时在对该字段处理得到string类型字段。这个显然太麻烦了而且不够优雅。

参考：MyBatis--BlobTypeHandler读取Blob类型字段

---

另外，话说回来我们存储的实际上是长文本，应该使用LONGTEXT更合适，这样中间就不需要转换了。

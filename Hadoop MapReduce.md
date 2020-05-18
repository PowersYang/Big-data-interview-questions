[toc]

## 1、手写WordCount

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.CombineTextInputFormat;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;


import java.io.IOException;

public class WordCount {
    public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {

        // 1、获取Job对象
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);

        // 2、设置jar存储位置
        job.setJarByClass(WordCount.class);

        // 3、关联Map和Reduce类
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);

        // 4、设置Mapper阶段输出的key和value类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);

        // 5、设置最终输出的key和value类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        // 如果不设置InputFormat，默认使用的是TextInputFormat.class
        job.setInputFormatClass(CombineTextInputFormat.class);
        // 虚拟存储切片最大值设置4M
        CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);

        // 6、设置输入路径和输出路径
        FileInputFormat.setInputPaths(job, new Path(args[0]));

        FileOutputFormat.setOutputPath(job, new Path(args[1]));


        // 7、提交job
        boolean result = job.waitForCompletion(true);

        System.exit(result ? 0 : 1);
    }

}


class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

        String line = value.toString();
        String[] words = line.split(" ");

        Text k = new Text();
        IntWritable v = new IntWritable(1);

        for (String word: words) {

            k.set(word);
            context.write(k, v);
        }
    }
}

class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {

        int sum = 0;
        for (IntWritable value: values) {
            sum += value.get();
        }

        context.write(key, new IntWritable(sum));
    }
}
```

## 2、
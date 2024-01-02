

```java
import com.mongodb.MongoClientSettings;
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoDatabase;
import io.circe.Encoder;
import io.circe.generic.semiauto.deriveEncoder;
import io.circe.java8.time.JavaTimeEncoders;
import io.circe.syntax.EncoderOps;
import org.bson.BsonReader;
import org.bson.BsonType;
import org.bson.BsonWriter;
import org.bson.codecs.Codec;
import org.bson.codecs.DecoderContext;
import org.bson.codecs.EncoderContext;
import org.bson.codecs.configuration.CodecConfigurationException;
import org.bson.codecs.configuration.CodecRegistries;
import org.bson.codecs.configuration.CodecRegistry;
import org.bson.codecs.configuration.Codecs;
import org.bson.codecs.configuration.CodecRegistries;
import org.mongodb.scala.MongoClient;
import org.mongodb.scala.bson.codecs.DEFAULT_CODEC_REGISTRY;
import org.mongodb.scala.bson.codecs.Macros;
import org.mongodb.scala.bson.codecs.ValueCodecProvider;
import org.mongodb.scala.bson.collection.immutable.Document;
import org.mongodb.scala.bson.codecs.Codec;
import org.mongodb.scala.bson.codecs.DocumentCodecProvider;
import org.mongodb.scala.bson.codecs.ValueCodecProvider;
import java.time.Instant;
import java.time.ZoneId;
import java.time.ZonedDateTime;
import java.util.concurrent.TimeUnit;

/** Custom codec for encoding and decoding ZonedDateTime instances zoned in UTC */
public class UTCZonedDateTimeMongoCodec implements Codec<ZonedDateTime> {
    @Override
    public ZonedDateTime decode(BsonReader reader, DecoderContext decoderContext) {
        if (reader.getCurrentBsonType() == BsonType.DATE_TIME) {
            return Instant.ofEpochMilli(reader.readDateTime()).atZone(ZoneId.of("UTC"));
        } else {
            throw new CodecConfigurationException("Could not decode into ZonedDateTime, expected DATE_TIME BsonType but got '" + reader.getCurrentBsonType() + "'.");
        }
    }

    @Override
    public void encode(BsonWriter writer, ZonedDateTime value, EncoderContext encoderContext) {
        try {
            writer.writeDateTime(value.withZoneSameInstant(ZoneId.of("UTC")).toInstant().toEpochMilli());
        } catch (ArithmeticException e) {
            throw new CodecConfigurationException("Unsupported LocalDateTime value '" + value + "' could not be converted to milliseconds: " + e.getMessage(), e);
        }
    }

    @Override
    public Class<ZonedDateTime> getEncoderClass() {
        return ZonedDateTime.class;
    }
}

/** Simple case class with a ZonedDateTime field */
public class Data {
    public final int id;
    public final String name;
    public final ZonedDateTime date;

    public Data(int id, String name, ZonedDateTime date) {
        this.id = id;
        this.name = name;
        this.date = date;
    }
}

/** Circe JSON encoder for our simple case class */
public class DataCirceEncoder {
    public static final Encoder<Data> dataCirceEncoder = deriveEncoder(Data.class);
}

/** Simple program for testing the codec */
public class TestProgram {
    public static void main(String[] args) {
        // Mongo codec registry which supports ZonedDateTime and Data instances
        CodecRegistry mongoCodecRegistry = CodecRegistries.fromRegistries(
            CodecRegistries.fromCodecs(new UTCZonedDateTimeMongoCodec()),
            CodecRegistries.fromProviders(Macros.createCodecProvider(Data.class)),
            DEFAULT_CODEC_REGISTRY
        );

        // Simple program for testing the codec
        MongoClient client = MongoClients.create("mongodb://localhost:27017");
        MongoDatabase db = client.getDatabase("test").withCodecRegistry(mongoCodecRegistry);
        MongoCollection<Data> col = db.getCollection("data", Data.class);

        // Example data
        ZonedDateTime testDataDate = ZonedDateTime.parse("1997-05-23T13:00:00+05:00[America/Bogota]");
        Data testData = new Data(3, "BalmungSan", testDataDate);

        // Delete existing documents and insert new data
        col.deleteMany(Document.parse("{}"));
        col.insertOne(testData);

        // Retrieve and print the result
        Data result = col.find().first();
        System.out.println(result);
    }
}
```
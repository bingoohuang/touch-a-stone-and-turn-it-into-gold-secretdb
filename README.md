# stonegold-secretdb
a demonstration for auto encryption/decryption of db table secret fields in internal training of touch-a-stone-and-turn-it-into-gold

1. initial code.

    ```java
    package stonegold;

    import java.sql.Connection;
    import java.sql.DriverManager;
    import java.sql.PreparedStatement;
    import java.sql.SQLException;

    public class Main {
        public static void main(String[] args) throws ClassNotFoundException, SQLException {
            Class.forName("org.h2.Driver");
            Connection conn = DriverManager.getConnection("jdbc:h2:~/stonegold", "sa", "");

            String createTableSql = "create table person(name varchar(10), id_no varchar(18), credit_card varchar(16))";
            PreparedStatement ps = conn.prepareStatement(createTableSql);
            ps.execute();
            ps.close();

            String insertSql = "insert into person(name, id_no, credit_card) values(?, ?, ?)";
            PreparedStatement insertPs = conn.prepareStatement(insertSql);
            insertPs.setString(1, "bingoo");
            insertPs.setString(2, "321421198312111234");
            insertPs.setString(3, "1111222233334444");
            insertPs.execute();
            insertPs.close();

            conn.close();
        }
    }
    ```

1. repeatable run.

    ```java
    String dropTableSql = "drop table if exists person";
    Statement statement = conn.createStatement();
    statement.execute(dropTableSql);
    statement.close();
    ```

1. use try-with-resources statement.

    ```java
    public static void main(String[] args) throws ClassNotFoundException, SQLException {
        Class.forName("org.h2.Driver");

        try (Connection conn = DriverManager.getConnection("jdbc:h2:~/stonegold", "sa", "")) {
            String dropTableSql = "drop table if exists person";
            try (Statement statement = conn.createStatement()) {
                statement.execute(dropTableSql);
            }

            String createTableSql = "create table  person(name varchar(10), id_no varchar(18), credit_card varchar(16))";
            try (PreparedStatement ps = conn.prepareStatement(createTableSql)) {
                ps.execute();
            }

            String insertSql = "insert into person(name, id_no, credit_card) values(?, ?, ?)";
            try (PreparedStatement insertPs = conn.prepareStatement(insertSql)) {
                insertPs.setString(1, "bingoo");
                insertPs.setString(2, "321421198312111234");
                insertPs.setString(3, "1111222233334444");
                insertPs.execute();
            }
        }
    }
    ```

1. verify the try-with-resources statement.

    ```java
    package stonegold;
    
    import java.lang.reflect.InvocationHandler;
    import java.lang.reflect.Method;
    import java.lang.reflect.Proxy;
    import java.sql.*;
    
    public class Main {
        public static void main(String[] args) throws ClassNotFoundException, SQLException {
            Class.forName("org.h2.Driver");
    
            try (Connection conn = DriverManager.getConnection("jdbc:h2:~/stonegold", "sa", "")) {
                String dropTableSql = "drop table if exists person";
                try (Statement statement = proxy(conn.createStatement())) {
                    statement.execute(dropTableSql);
                }
    
                String createTableSql = "create table  person(name varchar(10), id_no varchar(18), credit_card varchar(16))";
                try (PreparedStatement ps = proxy(conn.prepareStatement(createTableSql))) {
                    ps.execute();
                }
    
                String insertSql = "insert into person(name, id_no, credit_card) values(?, ?, ?)";
                try (PreparedStatement insertPs = proxy(conn.prepareStatement(insertSql))) {
                    insertPs.setString(1, "bingoo");
                    insertPs.setString(2, "321421198312111234");
                    insertPs.setString(3, "1111222233334444");
                    insertPs.execute();
                }
            }
        }
    
        private static PreparedStatement proxy(final PreparedStatement statement) {
            return (PreparedStatement) Proxy.newProxyInstance(Main.class.getClassLoader(),
                    new Class[]{PreparedStatement.class},
                    new InvocationHandler() {
                        @Override
                        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                            System.out.println(method.getName());
                            return method.invoke(statement, args);
                        }
                    });
        }
    
        private static Statement proxy(final Statement statement) {
            return (Statement) Proxy.newProxyInstance(Main.class.getClassLoader(),
                    new Class[]{Statement.class},
                    new InvocationHandler() {
                        @Override
                        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                            System.out.println(method.getName());
                            return method.invoke(statement, args);
                        }
                    });
        }
    
    }
    ```

1. try to proxy connection for sql parsing and auto encryption.

    ```java
    package stonegold;
    
    import com.google.common.collect.ImmutableSet;
    import com.google.common.primitives.Ints;
    
    import java.lang.reflect.InvocationHandler;
    import java.lang.reflect.Method;
    import java.lang.reflect.Proxy;
    import java.sql.*;
    
    public class Main {
        public static void main(String[] args) throws ClassNotFoundException, SQLException {
            Class.forName("org.h2.Driver");
    
            try (Connection conn = proxy(DriverManager.getConnection("jdbc:h2:~/stonegold", "sa", ""))) {
                String dropTableSql = "drop table if exists person";
                try (Statement statement = conn.createStatement()) {
                    statement.execute(dropTableSql);
                }
    
                String createTableSql = "create table  person(name varchar(10), id_no varchar(36), credit_card varchar(32))";
                try (PreparedStatement ps = conn.prepareStatement(createTableSql)) {
                    ps.execute();
                }
    
                String insertSql = "insert into person(name, id_no, credit_card) values(?, ?, ?)";
                try (PreparedStatement insertPs = conn.prepareStatement(insertSql)) {
                    insertPs.setString(1, "bingoo");
                    insertPs.setString(2, "321421198312111234");
                    insertPs.setString(3, "1111222233334444");
                    insertPs.execute();
                }
            }
        }
    
        private static Connection proxy(final Connection connection) {
            return (Connection) Proxy.newProxyInstance(Main.class.getClassLoader(),
                    new Class[]{Connection.class},
                    new InvocationHandler() {
                        @Override
                        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                            Object invoke = method.invoke(connection, args);
    
                            String methodName = method.getName();
                            if (!methodName.equals("prepareStatement")) return invoke;
    
                            String sql = (String) args[0];
                            SqlSecretFieldsParser sqlSecretFieldsParser = new SqlSecretFieldsParser();
                            final SecretFields secretFields = sqlSecretFieldsParser.parse(sql);
                            if (secretFields.isEmpty()) return invoke;
    
                            final PreparedStatement ps = (PreparedStatement) invoke;
                            return Proxy.newProxyInstance(Main.class.getClassLoader(),
                                    new Class[]{PreparedStatement.class},
                                    new InvocationHandler() {
                                        @Override
                                        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                                            String name = method.getName();
                                            if (!name.equals("setString")) return method.invoke(ps, args);
    
                                            if (!secretFields.isSecretField(args[0])) return method.invoke(ps, args);
    
                                            args[1] = "secret:" + args[1];
    
                                            return method.invoke(ps, args);
                                        }
                                    });
                        }
                    });
        }
    
        public static class SqlSecretFieldsParser {
            public SecretFields parse(String sql) {
                return new SecretFields(ImmutableSet.copyOf(Ints.asList(2, 3)));
            }
        }
        
        public static class SecretFields {
            private final ImmutableSet<Integer> secretFields;
    
            public SecretFields(ImmutableSet<Integer> secretFields) {
                this.secretFields = secretFields;
            }
    
            public boolean isEmpty() {
                return secretFields.isEmpty();
            }
    
            public boolean isSecretField(Object arg) {
                return secretFields.contains(arg);
            }
        }
    }
    ```



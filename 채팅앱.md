###### 이 채팅앱은 Firebase를 이용하여 사용자 인증, 실시간 데이터베이스 기능을 제공함.

___
### <span style = color:violet>코드 설명</span>
###### ChatActivity.java
- 채팅화면을 구성함
- 사용자 간의 메시지를 주고받을 수 있음
- `Firebase Realtime Database`에서 메시지를 읽고 씀
```java
package com.example.chatactivity;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.appcompat.widget.Toolbar;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import android.content.Intent;
import android.os.Bundle;
import android.view.Menu;
import android.view.MenuInflater;
import android.view.MenuItem;
import android.view.View;
import android.widget.EditText;
import android.widget.ImageView;
import android.widget.Toast;


import com.google.android.gms.tasks.OnFailureListener;
import com.google.android.gms.tasks.OnSuccessListener;
import com.google.firebase.auth.FirebaseAuth;
import com.google.firebase.database.DataSnapshot;
import com.google.firebase.database.DatabaseError;
import com.google.firebase.database.DatabaseReference;
import com.google.firebase.database.FirebaseDatabase;
import com.google.firebase.database.ValueEventListener;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

public class ChatActivity extends AppCompatActivity {
    String receiverId, receiverName, senderRoom, receiverRoom;
    String senderId, senderName;
    DatabaseReference dbReferenceSender, dbReferenceReceiver, userReference;
    ImageView sendBtn;
    EditText messageText;
    RecyclerView recyclerView;
    MessageAdapter messageAdapter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_chat);

        Toolbar toolbar = findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        userReference = FirebaseDatabase.getInstance().getReference("users");
        receiverId = getIntent().getStringExtra("id");
        receiverName = getIntent().getStringExtra("name");

        getSupportActionBar().setTitle(receiverName);
        if (receiverId != null) {
            senderRoom = FirebaseAuth.getInstance().getUid() + receiverId;
            receiverRoom = receiverId + FirebaseAuth.getInstance().getUid();
        }
        sendBtn = findViewById(R.id.sendMessageIcon);
        messageAdapter = new MessageAdapter(this);
        recyclerView = findViewById(R.id.chatrecycler);
        messageText = findViewById(R.id.messageEdit);

        recyclerView.setAdapter(messageAdapter);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        dbReferenceSender = FirebaseDatabase.getInstance().getReference("chats").child(senderRoom);
        dbReferenceReceiver = FirebaseDatabase.getInstance().getReference("chats").child(receiverRoom);

        dbReferenceSender.orderByChild("timestamp").addValueEventListener(new ValueEventListener() {
            @Override
            public void onDataChange(@NonNull DataSnapshot snapshot) {
                List<MessageModel> messages = new ArrayList<>();
                for (DataSnapshot dataSnapshot : snapshot.getChildren()) {
                    MessageModel messageModel = dataSnapshot.getValue(MessageModel.class);
                    messages.add(messageModel);
                }
                messageAdapter.clear();
                messageAdapter.addAll(messages);
                recyclerView.scrollToPosition(messageAdapter.getItemCount() - 1); // 최신 메시지로 스크롤
            }

            @Override
            public void onCancelled(@NonNull DatabaseError error) {

            }
        });


        sendBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                String message = messageText.getText().toString();
                if (message.trim().length() > 0) {
                    SendMessage(message);
                } else {
                    Toast.makeText(ChatActivity.this, "Message cannot be empty", Toast.LENGTH_SHORT).show();
                }
            }
        });

    }

    private void SendMessage(String message) {
        String messageId = UUID.randomUUID().toString();
        long timestamp = System.currentTimeMillis();
        MessageModel messageModel = new MessageModel(messageId, FirebaseAuth.getInstance().getUid(), message, timestamp);

        dbReferenceSender.child(messageId).setValue(messageModel)
                .addOnSuccessListener(new OnSuccessListener<Void>() {
                    @Override
                    public void onSuccess(Void unused) {
                        recyclerView.scrollToPosition(messageAdapter.getItemCount() - 1); // 최신 메시지로 스크롤
                    }
                })
                .addOnFailureListener(new OnFailureListener() {
                    @Override
                    public void onFailure(@NonNull Exception e) {
                        Toast.makeText(ChatActivity.this, "Failed to send message", Toast.LENGTH_SHORT).show();
                    }
                });
        dbReferenceReceiver.child(messageId).setValue(messageModel);
        messageText.setText("");
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        MenuInflater inflater = getMenuInflater();
        inflater.inflate(R.menu.main_menu, menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(@NonNull MenuItem item) {
        if(item.getItemId()==R.id.logout)
        {
            FirebaseAuth.getInstance().signOut();
            startActivity(new Intent(ChatActivity.this, SigninActivity.class));
            finish();
            return true;
        }
        return false;
    }
}
```

###### MainActivity.java
- 로그인 후 보이는 메인화면을 구성함
- 데이터베이스에서 사용자 목록을 가져와 `RecyclerView`에 표시함
```java
package com.example.chatactivity;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.appcompat.widget.Toolbar;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import android.content.Intent;
import android.os.Bundle;
import android.text.TextUtils;
import android.view.Menu;
import android.view.MenuInflater;
import android.view.MenuItem;
import android.view.View;
import android.widget.EditText;
import android.widget.TextView;

import com.google.firebase.auth.FirebaseAuth;
import com.google.firebase.database.DataSnapshot;
import com.google.firebase.database.DatabaseError;
import com.google.firebase.database.DatabaseReference;
import com.google.firebase.database.FirebaseDatabase;
import com.google.firebase.database.ValueEventListener;

import java.util.List;

public class MainActivity extends AppCompatActivity {

    RecyclerView recyclerView;
    UsersAdapter usersAdapter;
    String yourName;
    DatabaseReference databaseReference;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Toolbar toolbar = findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);
        String userName = getIntent().getStringExtra("user");
        getSupportActionBar().setTitle(userName);

        usersAdapter = new UsersAdapter(this);
        recyclerView = findViewById(R.id.recycler);

        recyclerView.setAdapter(usersAdapter);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));

        databaseReference = FirebaseDatabase.getInstance().getReference("users");
        databaseReference.addValueEventListener(new ValueEventListener() {
            @Override
            public void onDataChange(@NonNull DataSnapshot snapshot) {
                usersAdapter.clear();
                for(DataSnapshot dataSnapshot: snapshot.getChildren())
                {
                    String uId = dataSnapshot.getKey();
                    UserModel userModel = dataSnapshot.getValue(UserModel.class);
                    if(userModel!=null && userModel.getUserID()!=null && !userModel.getUserID().equals(FirebaseAuth.getInstance().getUid()))
                    {
                        usersAdapter.add(userModel);
                    }
                }
                List<UserModel> userModelList = usersAdapter.getUserModelList();
                usersAdapter.notifyDataSetChanged();

            }

            @Override
            public void onCancelled(@NonNull DatabaseError error) {

            }
        });



    }
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        MenuInflater inflater = getMenuInflater();
        inflater.inflate(R.menu.main_menu, menu);

        return true;
    }


    @Override
    public boolean onOptionsItemSelected(@NonNull MenuItem item) {
        if(item.getItemId()==R.id.logout)
        {
            FirebaseAuth.getInstance().signOut();
            startActivity(new Intent(MainActivity.this, SigninActivity.class));
            finish();
            return true;
        }

        return false;
    }
}
```
###### MessageAdapter.java
- 채팅 메시지를 RecyclerView에 표시하기 위한 어댑터
- 발신 메시지와 수신 메시지를 구분하여 각각 다른 레이아웃을 적용함
- add, addAll, clear 메소드로 메시지 데이터를 관리함
```java
package com.example.chatactivity;

import android.content.Context;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

import androidx.annotation.NonNull;
import androidx.recyclerview.widget.RecyclerView;

import com.google.firebase.auth.FirebaseAuth;

import java.util.ArrayList;
import java.util.List;

public class MessageAdapter extends RecyclerView.Adapter<MessageAdapter.MyViewHolder> {

    private static final int VIEW_TYPE_SENT = 1;
    private static final int VIEW_TYPE_RECEIVED = 2;
    private Context context;
    private List<MessageModel> messageModelList;

    public MessageAdapter(Context context) {
        this.context = context;
        this.messageModelList = new ArrayList<>();
    }

    public void add(MessageModel messageModel)
    {
        messageModelList.add(messageModel);
        notifyItemInserted(messageModelList.size() - 1);
    }

    public void addAll(List<MessageModel> messages) {
        messageModelList.addAll(messages);
        notifyDataSetChanged();
    }

    public void clear()
    {
        messageModelList.clear();
        notifyDataSetChanged();
    }

    @NonNull
    @Override
    public MessageAdapter.MyViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        if(viewType==VIEW_TYPE_SENT)
        {
            View view = inflater.inflate(R.layout.message_row_sent, parent, false);
            return new MyViewHolder(view);
        }
        else
        {
            View view = inflater.inflate(R.layout.message_row_received, parent, false);
            return new MyViewHolder(view);
        }
    }

    @Override
    public void onBindViewHolder(@NonNull MessageAdapter.MyViewHolder holder, int position)
    {
        MessageModel messageModel = messageModelList.get(position);
        if(messageModel.getSenderId().equals(FirebaseAuth.getInstance().getUid()))
        {
            holder.textViewSentMessage.setText(messageModel.getMessage());
        }
        else {
            holder.textViewReceivedMessage.setText(messageModel.getMessage());
        }

    }

    @Override
    public int getItemCount() {
        return messageModelList.size();
    }

    public List<MessageModel> getMessageModelList() {
        return messageModelList;
    }

    @Override
    public int getItemViewType(int position) {
        if(messageModelList.get(position).getSenderId().equals(FirebaseAuth.getInstance().getUid())) {
            return VIEW_TYPE_SENT;
        } else {
            return VIEW_TYPE_RECEIVED;
        }
    }

    public class MyViewHolder extends RecyclerView.ViewHolder{
        private TextView textViewSentMessage, textViewReceivedMessage;
        public MyViewHolder(@NonNull View itemView) {
            super(itemView);
            textViewSentMessage = itemView.findViewById(R.id.textViewSentMessage);
            textViewReceivedMessage = itemView.findViewById(R.id.textViewReceivedMessage);
        }
    }
}
```
###### MessageModel.java
- 채팅 메시지 데이터를 모델링함
- 메시지 ID, 발신자 ID, 메시지 내용, 타임스탬프를 포함
```java
package com.example.chatactivity;

public class MessageModel {

    private String messageId;
    private String senderId;
    private String message;
    private long timestamp;

    public MessageModel(String messageId, String senderId, String message, long timestamp) {
        this.messageId = messageId;
        this.senderId = senderId;
        this.message = message;
        this.timestamp = timestamp;
    }

    public MessageModel() {
    }

    public String getMessageId() {
        return messageId;
    }

    public String getSenderId() {
        return senderId;
    }

    public String getMessage() {
        return message;
    }

    public long getTimestamp() {
        return timestamp;
    }

    public void setMessageId(String messageId) {
        this.messageId = messageId;
    }

    public void setSenderId(String senderId) {
        this.senderId = senderId;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public void setTimestamp(long timestamp) {
        this.timestamp = timestamp;
    }
}
```

###### SigninActivity.java
- 사용자가 이메일과 비밀번호로 로그인할 수 있는 화면을 구성함
- Firebase Authentication을 이용하여 사용자 인증할 수 있음
- 로그인 성공시 MainActivity로 이동함
```java
package com.example.chatactivity;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;

import android.content.Intent;
import android.os.Bundle;
import android.text.TextUtils;
import android.view.View;
import android.widget.EditText;
import android.widget.TextView;
import android.widget.Toast;

import com.google.android.gms.tasks.OnFailureListener;
import com.google.android.gms.tasks.OnSuccessListener;
import com.google.firebase.auth.AuthResult;
import com.google.firebase.auth.FirebaseAuth;
import com.google.firebase.auth.FirebaseAuthInvalidUserException;
import com.google.firebase.database.DatabaseReference;
import com.google.firebase.database.FirebaseDatabase;

public class SigninActivity extends AppCompatActivity {

    EditText userEmail, userPassword;
    TextView signinBtn, signupBtn;
    String email, password;

    DatabaseReference databaseReference;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_signin);
        databaseReference = FirebaseDatabase.getInstance().getReference("users");
        userEmail = findViewById(R.id.emailtext);
        userPassword = findViewById(R.id.passwordtext);
        signinBtn = findViewById(R.id.login);
        signupBtn = findViewById(R.id.signup);

        signinBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                email = userEmail.getText().toString().trim();
                password = userPassword.getText().toString().trim();
                if (TextUtils.isEmpty(email)) {
                    userEmail.setError("Please enter your email");
                    userEmail.requestFocus();
                    return;
                }
                if (TextUtils.isEmpty(password)) {
                    userPassword.setError("Please enter your email");
                    userPassword.requestFocus();
                    return;
                }
                Signin();


            }
        });

        signupBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent(SigninActivity.this, SignupActivity.class);
                startActivity(intent);
            }
        });
    }

    private void Signin()
    {
        FirebaseAuth.getInstance().signInWithEmailAndPassword(email.trim(), password)
                .addOnSuccessListener(new OnSuccessListener<AuthResult>() {
                    @Override
                    public void onSuccess(AuthResult authResult) {
                        String username = FirebaseAuth.getInstance().getCurrentUser().getDisplayName();
                        Intent intent = new Intent(SigninActivity.this, MainActivity.class);
                        intent.putExtra("name", username);
                        startActivity(intent);
                        finish();
                    }
                })
                .addOnFailureListener(new OnFailureListener() {
                    @Override
                    public void onFailure(@NonNull Exception e) {
                        if(e instanceof FirebaseAuthInvalidUserException)
                        {
                            Toast.makeText(SigninActivity.this, "user does not exist", Toast.LENGTH_SHORT)
                                    .show();
                        }
                        else {
                            Toast.makeText(SigninActivity.this, "Authentication Falied", Toast.LENGTH_SHORT)
                                    .show();
                        }
                    }
                });
    }

    @Override
    protected void onStart() {
        super.onStart();
        if(FirebaseAuth.getInstance().getCurrentUser()!=null)
        {
            startActivity(new Intent(SigninActivity.this, MainActivity.class));
            finish();
        }
    }
}
```
###### SignupActivity.java
- 새로운 사용자가 회원가입할 수 있는 화면을 구성함
- Firebase Authentication을 이용하여 사용자를 등록할 수 있음
- 회원가입 성공 시 사용자 정보를 데이터베이스에 저장하고 MainActivity로 이동함
```java
package com.example.chatactivity;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;

import android.content.Intent;
import android.os.Bundle;
import android.text.TextUtils;
import android.view.View;
import android.widget.EditText;
import android.widget.TextView;
import android.widget.Toast;

import com.google.android.gms.tasks.OnFailureListener;
import com.google.android.gms.tasks.OnSuccessListener;
import com.google.firebase.auth.AuthResult;
import com.google.firebase.auth.FirebaseAuth;
import com.google.firebase.auth.FirebaseUser;
import com.google.firebase.auth.UserProfileChangeRequest;
import com.google.firebase.database.DatabaseReference;
import com.google.firebase.database.FirebaseDatabase;

public class SignupActivity extends AppCompatActivity {
    EditText userName, userEmail, userPassword;
    TextView signinBtn, signupBtn;
    String name, email, password;
    DatabaseReference databaseReference;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_signup);
        databaseReference = FirebaseDatabase.getInstance().getReference("users");
        userName = findViewById(R.id.usernametext);
        userEmail = findViewById(R.id.emailtext);
        userPassword = findViewById(R.id.passwordtext);
        signinBtn = findViewById(R.id.login);
        signupBtn = findViewById(R.id.signup);

        signupBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                name = userName.getText().toString().trim();
                email = userEmail.getText().toString().trim();
                password = userPassword.getText().toString().trim();
                if (TextUtils.isEmpty(name)) {
                    userEmail.setError("Please enter your name");
                    userEmail.requestFocus();
                    return;
                }
                if (TextUtils.isEmpty(email)) {
                    userEmail.setError("Please enter your email");
                    userEmail.requestFocus();
                    return;
                }
                if (TextUtils.isEmpty(password)) {
                    userPassword.setError("Please enter your email");
                    userPassword.requestFocus();
                    return;
                }
                Signup();

            }
        });

        signinBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent(SignupActivity.this, SigninActivity.class);
                startActivity(intent);
            }
        });
    }

    protected void onStart() {
        super.onStart();
        if(FirebaseAuth.getInstance().getCurrentUser()!=null)
        {
            startActivity(new Intent(SignupActivity.this, MainActivity.class));
            finish();
        }
    }

    private void Signup() {
        FirebaseAuth.getInstance().createUserWithEmailAndPassword(email.trim(), password)
                .addOnSuccessListener(new OnSuccessListener<AuthResult>() {
                    @Override
                    public void onSuccess(AuthResult authResult) {
                        UserProfileChangeRequest userProfileChangeRequest = new UserProfileChangeRequest.Builder().setDisplayName(name).build();
                        FirebaseUser firebaseUser = FirebaseAuth.getInstance().getCurrentUser();
                        firebaseUser.updateProfile(userProfileChangeRequest);

                        UserModel userModel = new UserModel(FirebaseAuth.getInstance().getUid(), name, email, password);
                        databaseReference.child(FirebaseAuth.getInstance().getUid()).setValue(userModel);
                        Intent intent = new Intent(SignupActivity.this, MainActivity.class);
                        intent.putExtra("name", name);
                        startActivity(intent);
                        finish();
                    }
                })
                .addOnFailureListener(new OnFailureListener() {
                    @Override
                    public void onFailure(@NonNull Exception e) {
                       Toast.makeText(SignupActivity.this, "Signup Failed", Toast.LENGTH_SHORT).show();
                    }
                });


    }
}
```
###### UserModel.java
- 사용자 목록을 RecyclerView에 표시하기 위한 어댑터
- 사용자 항목을 클릭하면 해당 사용자와의 ChatActivity로 이동함
```java
package com.example.chatactivity;

public class UserModel {

    String userID;
    String userName;
    String userEmail;
    String userPassword;


    public UserModel() {
    }

    public UserModel(String userID, String userName, String userEmail, String userPassword) {
        this.userID = userID;
        this.userName = userName;
        this.userEmail = userEmail;
        this.userPassword = userPassword;
    }

    public String getUserID() {
        return userID;
    }

    public String getUserName() {
        return userName;
    }

    public String getUserEmail() {
        return userEmail;
    }

    public String getUserPassword() {
        return userPassword;
    }

    public void setUserID(String userID) {
        this.userID = userID;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public void setUserEmail(String userEmail) {
        this.userEmail = userEmail;
    }

    public void setUserPassword(String userPassword) {
        this.userPassword = userPassword;
    }
}
```
###### activity_chat.xml
- 채팅화면 레이아웃
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ChatActivity"
    android:background="@color/colorBackground">

    <androidx.appcompat.widget.Toolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="@color/colorLightBlue"
        app:titleTextColor="@color/white"/>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/chatrecycler"
        android:layout_below="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_above="@+id/sendmessageLayout"
        android:padding="8dp"/>

    <RelativeLayout
        android:id="@+id/sendmessageLayout"
        android:layout_alignParentBottom="true"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="8dp"
        android:background="@color/white">

        <EditText
            android:id="@+id/messageEdit"
            android:hint="Write your message here"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_toStartOf="@+id/sendMessageIcon"
            android:layout_alignParentLeft="true"
            android:padding="16dp"
            android:background="@drawable/edit_style_box"
            android:layout_weight="1"/>

        <ImageView
            android:id="@+id/sendMessageIcon"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:layout_alignParentRight="true"
            android:padding="8dp"
            android:src="@drawable/send_icon"/>
    </RelativeLayout>
</RelativeLayout>
```
###### activity_main.xml
- 메인 화면 레이아웃
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:background="@color/colorBackground"
    tools:context=".MainActivity">

  <androidx.appcompat.widget.Toolbar
      android:id="@+id/toolbar"
      android:layout_width="match_parent"
      android:layout_height="?android:attr/actionBarSize"
      android:background="@color/colorLightBlue"
      app:titleTextColor="@color/white"/>

  <androidx.recyclerview.widget.RecyclerView
      android:id="@+id/recycler"
      android:layout_width="match_parent"
      android:layout_height="wrap_content"
      android:padding="8dp"/>
</LinearLayout>
```
###### activity_signin.xml
- 로그인 화면 레이아웃
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp"
    android:background="@color/colorBackground"
    tools:context=".SigninActivity">

    <ImageView
        android:id="@+id/image"
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:layout_centerHorizontal="true"
        android:src="@drawable/logo"
        android:layout_marginBottom="32dp"/>

    <EditText
        android:id="@+id/emailtext"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Email"
        android:layout_below="@+id/image"
        android:layout_marginVertical="8dp"
        android:inputType="textEmailAddress"
        android:background="@drawable/edit_style_box"
        android:padding="16dp"/>

    <EditText
        android:id="@+id/passwordtext"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Password"
        android:layout_below="@+id/emailtext"
        android:layout_marginVertical="8dp"
        android:inputType="textPassword"
        android:background="@drawable/edit_style_box"
        android:padding="16dp"/>

    <TextView
        android:id="@+id/login"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_below="@+id/passwordtext"
        android:text="로그인"
        android:textSize="24sp"
        android:gravity="center"
        android:padding="16dp"
        android:background="@drawable/button_background"
        android:textColor="@color/white"
        android:layout_marginVertical="16dp"/>

    <TextView
        android:id="@+id/signup"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_below="@+id/login"
        android:text="회원가입"
        android:textColor="@color/white"
        android:textSize="24sp"
        android:gravity="center"
        android:padding="16dp"
        android:background="@drawable/button_background"
        android:layout_marginVertical="16dp"/>
</RelativeLayout>
```
###### activity_signup.xml
- 회원가입 화면 레이아웃
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:padding="16dp"
    android:background="@color/colorBackground"
    tools:context=".SignupActivity">

    <ImageView
        android:id="@+id/image"
        android:layout_width="200dp"
        android:layout_height="200dp"
        android:layout_centerHorizontal="true"
        android:src="@drawable/logo"
        android:layout_marginBottom="32dp"/>

    <EditText
        android:id="@+id/usernametext"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="User name"
        android:layout_below="@+id/image"
        android:layout_marginVertical="8dp"
        android:inputType="text"
        android:background="@drawable/edit_style_box"
        android:padding="16dp"/>

    <EditText
        android:id="@+id/emailtext"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Email"
        android:layout_below="@+id/usernametext"
        android:layout_marginVertical="8dp"
        android:inputType="textEmailAddress"
        android:background="@drawable/edit_style_box"
        android:padding="16dp"/>

    <EditText
        android:id="@+id/passwordtext"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Password"
        android:layout_below="@+id/emailtext"
        android:layout_marginVertical="8dp"
        android:inputType="textPassword"
        android:background="@drawable/edit_style_box"
        android:padding="16dp"/>

    <TextView
        android:id="@+id/signup"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_below="@+id/passwordtext"
        android:text="가입하기"
        android:textSize="24sp"
        android:gravity="center"
        android:padding="16dp"
        android:background="@drawable/button_background"
        android:textColor="@color/white"
        android:layout_marginVertical="16dp"/>

    <TextView
        android:id="@+id/login"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_below="@+id/signup"
        android:text="로그인 화면"
        android:background="@drawable/button_background"
        android:textColor="@color/white"
        android:textSize="24sp"
        android:gravity="center"
        android:padding="16dp"
        android:layout_marginVertical="16dp"/>
</RelativeLayout>
```
###### message_row_received.xml
- 수신 메시지 레이아웃
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:gravity="start"
    android:padding="8dp">

    <RelativeLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:padding="12dp"
        android:layout_margin="8dp"
        android:background="@drawable/rounded_corner_received">

        <TextView
            android:id="@+id/textViewReceivedMessage"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="@color/colorText"
            android:textSize="16sp"/>
    </RelativeLayout>
</LinearLayout>
```
###### message_row_sent.xml
- 발신 메시지 레이아웃
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:gravity="end"
    android:padding="8dp">

    <RelativeLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:padding="12dp"
        android:layout_margin="8dp"
        android:background="@drawable/rounded_corner_sent">

        <TextView
            android:id="@+id/textViewSentMessage"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="@color/colorText"
            android:textSize="16sp"/>
    </RelativeLayout>
</LinearLayout>
```
###### button_background.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_pressed="true">
        <shape android:shape="rectangle">
            <corners android:radius="8dp" />
            <solid android:color="@color/PrimaryDark" />
        </shape>
    </item>
    <item>
        <shape android:shape="rectangle">
            <corners android:radius="8dp"/>
            <solid android:color="@color/Primary"/>
        </shape>
    </item>
</selector>
```
###### edit_style_box.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">

    <solid android:color="#FFFFFF" />

    <coners android:radius="8dp" />
    <stroke
        android:width="1dp"
        android:color="#CCCCCC" />
</shape>
```
###### colors.xml
```xml
<resources>
    <color name="colorPrimary">#6200EE</color>
    <color name="colorPrimaryDark">#3700B3</color>
    <color name="colorAccent">#03DAC5</color>
    <color name="colorBackground">#F5F5F5</color>
    <color name="colorMessageReceived">#E0E0E0</color>
    <color name="colorMessageSent">#DCF8C6</color>
    <color name="colorText">#000000</color>
    <color name="black">#FF000000</color>
    <color name="white">#FFFFFFFF</color>
    <color name="PrimaryDark">#075e54</color>
    <color name="Primary">#128c7e</color>
    <color name="colorLightBlue">#ADD8E6</color>
</resources>
```
###### strings.xml
```xml
<resources>
    <string name="app_name">ChatActivity</string>
</resources>
```

___
### <span style = color:violet>프로젝트 실행</span>
###### 회원가입 화면
![스크린샷 2024-06-19 182130](https://github.com/HeungJunBag/AndroidProgramming/assets/169026858/a173f9f6-0524-4462-99b0-6a9f622a8762)


###### 로그인 화면
![스크린샷 2024-06-19 182630](https://github.com/HeungJunBag/AndroidProgramming/assets/169026858/1b2f1eac-e6f7-485a-80c7-3be7adcb0e35)



###### 등록된 사용자 목록 
![스크린샷 2024-06-19 182808](https://github.com/HeungJunBag/AndroidProgramming/assets/169026858/173e2b90-34d8-4de8-8cf4-816e4ad902b7)



###### 채팅화면
![스크린샷 2024-06-19 183227](https://github.com/HeungJunBag/AndroidProgramming/assets/169026858/2e955fca-c1ad-4fff-a2fe-09c9d4e602c1)


###### 데이터베이스
![스크린샷 2024-06-19 183402](https://github.com/HeungJunBag/AndroidProgramming/assets/169026858/a99eebcd-0470-4bde-92c4-5191bd615fb2)



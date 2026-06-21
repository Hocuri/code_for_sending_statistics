# Code I Wrote for the Option to Send Statistics in Delta Chat

All of this code was written by me except where noted otherwise, then the Delta Chat and Chatmail maintainers reviewed it, and I addressed their comments.

1. [Delta Chat for Android](#delta-chat-for-android)
2. [Chatmail Core](#chatmail-core)
3. [Bot that collects the statistics](#bot-that-collects-the-statistics)
4. [Evaluation scripts for the quality of the data collected in the pre-study](#evaluation-scripts-for-the-quality-of-the-data-collected-in-the-pre-study)

## Delta Chat for Android

I implemented the UI part of statistics-sending in https://github.com/deltachat/deltachat-android/. The diff of my contribution is:

```diff
diff --git a/CHANGELOG.md b/CHANGELOG.md
index f4d9dd544..318d24e9f 100644
--- a/CHANGELOG.md
+++ b/CHANGELOG.md
@@ -11,6 +11,7 @@
 * Target Android 16
 * Change color of links in text messages
 * Improve edge-to-edge support
+* Add the option (opt-in) to send anonymous statistics to Delta Chat's developers
 * Metadata protection: protect message recipients
 * Allow to withdraw channel invite links and QR codes
 * Allow to open externally links clicked inside in-chat apps
diff --git a/src/main/java/org/thoughtcrime/securesms/ConversationFragment.java b/src/main/java/org/thoughtcrime/securesms/ConversationFragment.java
index 02f2f1271..3929de365 100644
--- a/src/main/java/org/thoughtcrime/securesms/ConversationFragment.java
+++ b/src/main/java/org/thoughtcrime/securesms/ConversationFragment.java
@@ -744,6 +744,9 @@ public class ConversationFragment extends MessageSelectorFragment
             else if(DozeReminder.isDozeReminderMsg(getContext(), messageRecord)) {
                 DozeReminder.dozeReminderTapped(getContext());
             }
+            else if(StatsSending.isStatsSendingDeviceMsg(getContext(), messageRecord)) {
+                StatsSending.statsDeviceMsgTapped(getActivity());
+            }
             else if(messageRecord.getInfoType() == DcMsg.DC_INFO_WEBXDC_INFO_MESSAGE) {
                 if (messageRecord.getParent() != null) {
                     // if the parent webxdc message still exists
diff --git a/src/main/java/org/thoughtcrime/securesms/ConversationListActivity.java b/src/main/java/org/thoughtcrime/securesms/ConversationListActivity.java
index 1076889db..f0fa56c83 100644
--- a/src/main/java/org/thoughtcrime/securesms/ConversationListActivity.java
+++ b/src/main/java/org/thoughtcrime/securesms/ConversationListActivity.java
@@ -80,6 +80,8 @@ import org.thoughtcrime.securesms.util.SendRelayedMessageUtil;
 import org.thoughtcrime.securesms.util.StorageUtil;
 import org.thoughtcrime.securesms.util.Util;
 import org.thoughtcrime.securesms.util.ViewUtil;
+import chat.delta.rpc.types.SecurejoinSource;
+import chat.delta.rpc.types.SecurejoinUiPath;
 
 import java.util.ArrayList;
 import java.util.Date;
@@ -489,7 +491,7 @@ public class ConversationListActivity extends PassphraseRequiredActionBarActivit
 
       if (uri.getScheme().equalsIgnoreCase(OPENPGP4FPR) || Util.isInviteURL(uri)) {
         QrCodeHandler qrCodeHandler = new QrCodeHandler(this);
-        qrCodeHandler.handleQrData(uri.toString());
+        qrCodeHandler.handleQrData(uri.toString(), SecurejoinSource.ExternalLink, null);
       }
     }
   }
@@ -571,7 +573,7 @@ public class ConversationListActivity extends PassphraseRequiredActionBarActivit
       case IntentIntegrator.REQUEST_CODE:
         IntentResult scanResult = IntentIntegrator.parseActivityResult(requestCode, resultCode, data);
         QrCodeHandler qrCodeHandler = new QrCodeHandler(this);
-        qrCodeHandler.onScanPerformed(scanResult);
+        qrCodeHandler.onScanPerformed(scanResult, SecurejoinUiPath.QrIcon);
         break;
       default:
         break;
diff --git a/src/main/java/org/thoughtcrime/securesms/NewConversationActivity.java b/src/main/java/org/thoughtcrime/securesms/NewConversationActivity.java
index 2421ef92d..628525e37 100644
--- a/src/main/java/org/thoughtcrime/securesms/NewConversationActivity.java
+++ b/src/main/java/org/thoughtcrime/securesms/NewConversationActivity.java
@@ -39,6 +39,8 @@ import org.thoughtcrime.securesms.qr.QrActivity;
 import org.thoughtcrime.securesms.qr.QrCodeHandler;
 import org.thoughtcrime.securesms.util.MailtoUtil;
 
+import chat.delta.rpc.types.SecurejoinUiPath;
+
 /**
  * Activity container for starting a new conversation.
  *
@@ -137,7 +139,7 @@ public class NewConversationActivity extends ContactSelectionActivity {
       case IntentIntegrator.REQUEST_CODE:
         IntentResult scanResult = IntentIntegrator.parseActivityResult(requestCode, resultCode, data);
         QrCodeHandler qrCodeHandler = new QrCodeHandler(this);
-        qrCodeHandler.onScanPerformed(scanResult);
+        qrCodeHandler.onScanPerformed(scanResult, SecurejoinUiPath.NewContact);
         break;
       default:
         break;
diff --git a/src/main/java/org/thoughtcrime/securesms/StatsSending.java b/src/main/java/org/thoughtcrime/securesms/StatsSending.java
new file mode 100644
index 000000000..4b98dfbc0
--- /dev/null
+++ b/src/main/java/org/thoughtcrime/securesms/StatsSending.java
@@ -0,0 +1,100 @@
+package org.thoughtcrime.securesms;
+
+import static org.thoughtcrime.securesms.connect.DcHelper.CONFIG_STATS_SENDING;
+import static org.thoughtcrime.securesms.connect.DcHelper.openHelp;
+
+import android.app.Activity;
+import android.content.Context;
+import android.text.util.Linkify;
+import android.widget.TextView;
+
+import androidx.appcompat.app.AlertDialog;
+
+import com.b44t.messenger.DcContact;
+import com.b44t.messenger.DcContext;
+import com.b44t.messenger.DcMsg;
+
+import org.thoughtcrime.securesms.connect.DcHelper;
+import org.thoughtcrime.securesms.util.IntentUtils;
+import org.thoughtcrime.securesms.util.Prefs;
+
+import java.util.Locale;
+
+public class StatsSending {
+  /** @noinspection unused: We will start adding a device message once stats-sending is tested a bit */
+  public static void maybeAddStatsSendingDeviceMsg(Context context) {
+    if (Prefs.getStatsDeviceMsgId(context) != 0) {
+      return;
+    }
+    if (DcHelper.getInt(context, CONFIG_STATS_SENDING) != 0) {
+      return; // Stats-sending is already enabled
+    }
+    DcContext dcContext = DcHelper.getContext(context);
+    DcMsg msg = new DcMsg(dcContext, DcMsg.DC_MSG_TEXT);
+    msg.setText(context.getString(R.string.stats_device_message));
+    int msgId = dcContext.addDeviceMsg("stats_device_message", msg);
+    if (msgId!=0) {
+      Prefs.setStatsDeviceMsgId(context, msgId);
+    }
+  }
+
+  public static boolean isStatsSendingDeviceMsg(Context context, DcMsg msg) {
+    return msg != null
+      && msg.getFromId() == DcContact.DC_CONTACT_ID_DEVICE
+      && msg.getId() == Prefs.getStatsDeviceMsgId(context);
+  }
+
+  public static void statsDeviceMsgTapped(Activity activity) {
+    if (DcHelper.getInt(activity, CONFIG_STATS_SENDING) != 0) {
+      showStatsDisableDialog(activity);
+    } else {
+      showStatsConfirmationDialog(activity, () -> {});
+    }
+  }
+
+  public static void showStatsConfirmationDialog(Activity activity, Runnable onConfigChangedListener) {
+    AlertDialog d = new AlertDialog.Builder(activity)
+      .setMessage(R.string.stats_confirmation_dialog)
+      .setNegativeButton(R.string.cancel, (_d, i) -> {})
+      .setPositiveButton(R.string.yes, (_d, i) -> {
+        DcHelper.set(activity, DcHelper.CONFIG_STATS_SENDING, "1");
+        onConfigChangedListener.run();
+        showStatsThanksDialog(activity);
+      })
+      .setNeutralButton(R.string.more_info_desktop, (_d, i) -> openHelp(activity, "#statssending"))
+      .create();
+    d.show();
+    try {
+      //noinspection DataFlowIssue
+      Linkify.addLinks((TextView) d.findViewById(android.R.id.message), Linkify.WEB_URLS);
+    } catch (NullPointerException e) {
+      //noinspection CallToPrintStackTrace
+      e.printStackTrace();
+    }
+  }
+
+  private static void showStatsThanksDialog(Activity activity) {
+    String stats_id = DcHelper.get(activity, DcHelper.CONFIG_STATS_ID);
+    new AlertDialog.Builder(activity)
+      .setMessage(R.string.stats_thanks)
+      .setNeutralButton(R.string.no, (d, i) -> {})
+      .setPositiveButton(R.string.yes, (d, i) -> {
+        String ln = Locale.getDefault().getLanguage();
+        IntentUtils.showInBrowser(activity, "https://cispa.qualtrics.com/jfe/form/SV_9YmhkpGa48KxfLg?id=" + stats_id + "&ln=" + ln);
+      })
+      .show();
+  }
+
+  public static void showStatsDisableDialog(Activity activity) {
+    AlertDialog d = new AlertDialog.Builder(activity)
+      .setMessage(R.string.stats_disable_dialog)
+      .setNegativeButton(R.string.disable, (_d, i) -> {
+        DcHelper.set(activity, DcHelper.CONFIG_STATS_SENDING, "0");
+      })
+      .setPositiveButton(R.string.stats_keep_sending, (_d, i) -> {
+      })
+      .setNeutralButton(R.string.more_info_desktop, (_d, i) -> openHelp(activity, "#statssending"))
+      .create();
+    d.show();
+  }
+}
diff --git a/src/main/java/org/thoughtcrime/securesms/WebViewActivity.java b/src/main/java/org/thoughtcrime/securesms/WebViewActivity.java
index b092e3c62..2d97e16c0 100644
--- a/src/main/java/org/thoughtcrime/securesms/WebViewActivity.java
+++ b/src/main/java/org/thoughtcrime/securesms/WebViewActivity.java
@@ -28,6 +28,8 @@ import org.thoughtcrime.securesms.util.IntentUtils;
 import org.thoughtcrime.securesms.util.Util;
 import org.thoughtcrime.securesms.util.ViewUtil;
 
+import chat.delta.rpc.types.SecurejoinSource;
+
 import java.net.IDN;
 
 public class WebViewActivity extends PassphraseRequiredActionBarActivity
@@ -302,7 +304,7 @@ public class WebViewActivity extends PassphraseRequiredActionBarActivity
     // invite-links should be handled directly
     String schema = url.split(":")[0].toLowerCase();
     if (schema.equals("openpgp4fpr") || url.startsWith("https://" + Util.INVITE_DOMAIN + "/")) {
-      new QrCodeHandler(this).handleQrData(url);
+      new QrCodeHandler(this).handleQrData(url, SecurejoinSource.InternalLink, null);
       return true; // abort internal loading
     }
 
diff --git a/src/main/java/org/thoughtcrime/securesms/connect/DcHelper.java b/src/main/java/org/thoughtcrime/securesms/connect/DcHelper.java
index 2f0b6163e..824cd1f84 100644
--- a/src/main/java/org/thoughtcrime/securesms/connect/DcHelper.java
+++ b/src/main/java/org/thoughtcrime/securesms/connect/DcHelper.java
@@ -62,6 +62,8 @@ public class DcHelper {
     public static final String CONFIG_PROXY_URL = "proxy_url";
     public static final String CONFIG_WEBXDC_REALTIME_ENABLED = "webxdc_realtime_enabled";
     public static final String CONFIG_PRIVATE_TAG = "private_tag";
+    public static final String CONFIG_STATS_SENDING = "stats_sending";
+    public static final String CONFIG_STATS_ID = "stats_id";
 
     public static DcContext getContext(@NonNull Context context) {
         return ApplicationContext.getInstance(context).dcContext;
diff --git a/src/main/java/org/thoughtcrime/securesms/contacts/NewContactActivity.java b/src/main/java/org/thoughtcrime/securesms/contacts/NewContactActivity.java
index a9b11e960..494c8655a 100644
--- a/src/main/java/org/thoughtcrime/securesms/contacts/NewContactActivity.java
+++ b/src/main/java/org/thoughtcrime/securesms/contacts/NewContactActivity.java
@@ -22,6 +22,8 @@ import org.thoughtcrime.securesms.connect.DcHelper;
 import org.thoughtcrime.securesms.qr.QrCodeHandler;
 import org.thoughtcrime.securesms.util.ViewUtil;
 
+import chat.delta.rpc.types.SecurejoinUiPath;
+
 public class NewContactActivity extends PassphraseRequiredActionBarActivity
 {
 
@@ -105,7 +107,7 @@ public class NewContactActivity extends PassphraseRequiredActionBarActivity
     if (requestCode == IntentIntegrator.REQUEST_CODE) {
       IntentResult scanResult = IntentIntegrator.parseActivityResult(requestCode, resultCode, data);
       QrCodeHandler qrCodeHandler = new QrCodeHandler(this);
-      qrCodeHandler.onScanPerformed(scanResult);
+      qrCodeHandler.onScanPerformed(scanResult, SecurejoinUiPath.NewContact);
     }
   }
 }
diff --git a/src/main/java/org/thoughtcrime/securesms/preferences/AdvancedPreferenceFragment.java b/src/main/java/org/thoughtcrime/securesms/preferences/AdvancedPreferenceFragment.java
index 1b8de05a4..8fe732761 100644
--- a/src/main/java/org/thoughtcrime/securesms/preferences/AdvancedPreferenceFragment.java
+++ b/src/main/java/org/thoughtcrime/securesms/preferences/AdvancedPreferenceFragment.java
@@ -5,6 +5,7 @@ import static android.text.InputType.TYPE_TEXT_VARIATION_URI;
 import static org.thoughtcrime.securesms.connect.DcHelper.CONFIG_BCC_SELF;
 import static org.thoughtcrime.securesms.connect.DcHelper.CONFIG_MVBOX_MOVE;
 import static org.thoughtcrime.securesms.connect.DcHelper.CONFIG_ONLY_FETCH_MVBOX;
+import static org.thoughtcrime.securesms.connect.DcHelper.CONFIG_STATS_SENDING;
 import static org.thoughtcrime.securesms.connect.DcHelper.CONFIG_SHOW_EMAILS;
 import static org.thoughtcrime.securesms.connect.DcHelper.CONFIG_WEBXDC_REALTIME_ENABLED;
 
@@ -16,6 +17,7 @@ import android.os.Bundle;
 import android.util.Log;
 import android.view.View;
 import android.widget.EditText;
+import android.widget.TextView;
 import android.widget.Toast;
 
 import androidx.annotation.NonNull;
@@ -29,8 +31,10 @@ import org.thoughtcrime.securesms.ApplicationPreferencesActivity;
 import org.thoughtcrime.securesms.LogViewActivity;
 import org.thoughtcrime.securesms.R;
 import org.thoughtcrime.securesms.relay.RelayListActivity;
+import org.thoughtcrime.securesms.StatsSending;
 import org.thoughtcrime.securesms.connect.DcEventCenter;
 import org.thoughtcrime.securesms.proxy.ProxySettingsActivity;
+import org.thoughtcrime.securesms.util.IntentUtils;
 import org.thoughtcrime.securesms.util.Prefs;
 import org.thoughtcrime.securesms.util.StreamUtil;
 import org.thoughtcrime.securesms.util.Util;
@@ -49,6 +53,7 @@ public class AdvancedPreferenceFragment extends ListSummaryPreferenceFragment
   private static final String TAG = AdvancedPreferenceFragment.class.getSimpleName();
 
   private ListPreference showEmails;
+  CheckBoxPreference selfReportingCheckbox;
   CheckBoxPreference multiDeviceCheckbox;
   CheckBoxPreference mvboxMoveCheckbox;
   CheckBoxPreference onlyFetchMvboxCheckbox;
@@ -191,6 +196,22 @@ public class AdvancedPreferenceFragment extends ListSummaryPreferenceFragment
       });
     }
 
+    selfReportingCheckbox = this.findPreference("pref_stats_sending");
+    if (selfReportingCheckbox != null) {
+      selfReportingCheckbox.setOnPreferenceChangeListener((preference, newValue) -> {
+        boolean enabled = (Boolean) newValue;
+        if (enabled) {
+          StatsSending.showStatsConfirmationDialog(requireActivity(), () -> {
+            ((CheckBoxPreference)preference).setChecked(true);
+          });
+          return false;
+        } else {
+          dcContext.setConfigInt(CONFIG_STATS_SENDING, 0);
+          return true;
+        }
+      });
+    }
+
     Preference proxySettings = this.findPreference("proxy_settings_button");
     if (proxySettings != null) {
       proxySettings.setOnPreferenceClickListener((preference) -> {
@@ -226,6 +247,7 @@ public class AdvancedPreferenceFragment extends ListSummaryPreferenceFragment
     showEmails.setValue(value);
     updateListSummary(showEmails, value);
 
+    selfReportingCheckbox.setChecked(0!=dcContext.getConfigInt(CONFIG_STATS_SENDING));
     multiDeviceCheckbox.setChecked(0!=dcContext.getConfigInt(CONFIG_BCC_SELF));
     mvboxMoveCheckbox.setChecked(0!=dcContext.getConfigInt(CONFIG_MVBOX_MOVE));
     onlyFetchMvboxCheckbox.setChecked(0!=dcContext.getConfigInt(CONFIG_ONLY_FETCH_MVBOX));
diff --git a/src/main/java/org/thoughtcrime/securesms/qr/QrActivity.java b/src/main/java/org/thoughtcrime/securesms/qr/QrActivity.java
index 6be2b363e..1b62f61e1 100644
--- a/src/main/java/org/thoughtcrime/securesms/qr/QrActivity.java
+++ b/src/main/java/org/thoughtcrime/securesms/qr/QrActivity.java
@@ -2,6 +2,7 @@ package org.thoughtcrime.securesms.qr;
 
 import android.Manifest;
 import android.app.Activity;
+import android.content.ComponentName;
 import android.content.Intent;
 import android.content.pm.PackageManager;
 import android.graphics.Bitmap;
@@ -29,6 +30,8 @@ import com.google.zxing.Result;
 import com.google.zxing.common.HybridBinarizer;
 
 import org.thoughtcrime.securesms.BaseActionBarActivity;
+import org.thoughtcrime.securesms.ConversationListActivity;
+import org.thoughtcrime.securesms.NewConversationActivity;
 import org.thoughtcrime.securesms.R;
 import org.thoughtcrime.securesms.connect.DcHelper;
 import org.thoughtcrime.securesms.contacts.NewContactActivity;
@@ -41,6 +44,9 @@ import org.thoughtcrime.securesms.util.ViewUtil;
 import java.io.FileNotFoundException;
 import java.io.InputStream;
 
+import chat.delta.rpc.types.SecurejoinSource;
+import chat.delta.rpc.types.SecurejoinUiPath;
+
 public class QrActivity extends BaseActionBarActivity implements View.OnClickListener {
 
     private final static String TAG = QrActivity.class.getSimpleName();
@@ -149,7 +155,7 @@ public class QrActivity extends BaseActionBarActivity implements View.OnClickLis
         AttachmentManager.selectImage(this, REQUEST_CODE_IMAGE);
       } else if (itemId == R.id.paste) {
         QrCodeHandler qrCodeHandler = new QrCodeHandler(this);
-        qrCodeHandler.handleQrData(Util.getTextFromClipboard(this));
+        qrCodeHandler.handleQrData(Util.getTextFromClipboard(this), SecurejoinSource.Clipboard, getUiPath());
       }
 
         return false;
@@ -200,7 +206,7 @@ public class QrActivity extends BaseActionBarActivity implements View.OnClickLis
                         try {
                             Result result = reader.decode(bBitmap);
                             QrCodeHandler qrCodeHandler = new QrCodeHandler(this);
-                            qrCodeHandler.handleQrData(result.getText());
+                            qrCodeHandler.handleQrData(result.getText(), SecurejoinSource.ImageLoaded, getUiPath());
                         } catch (NotFoundException e) {
                             Log.e(TAG, "decode exception", e);
                             Toast.makeText(this, getString(R.string.qrscan_failed), Toast.LENGTH_LONG).show();
@@ -213,6 +219,21 @@ public class QrActivity extends BaseActionBarActivity implements View.OnClickLis
         }
     }
 
+    private SecurejoinUiPath getUiPath() {
+        SecurejoinUiPath uiPath = null;
+        ComponentName caller = this.getCallingActivity();
+        if (caller != null) {
+            if (caller.getClassName().equals(NewConversationActivity.class.getName())) {
+                uiPath = SecurejoinUiPath.NewContact;
+            } else if (caller.getClassName().equals(ConversationListActivity.class.getName())
+                    // RoutingActivity is an alias for ConversationListActivity
+                    || caller.getClassName().endsWith(".RoutingActivity")) {
+                uiPath = SecurejoinUiPath.QrIcon;
+            }
+        }
+        return uiPath;
+    }
+
     @Override
     public void onClick(View v) {
         viewPager.setCurrentItem(TAB_SCAN);
diff --git a/src/main/java/org/thoughtcrime/securesms/qr/QrCodeHandler.java b/src/main/java/org/thoughtcrime/securesms/qr/QrCodeHandler.java
index 46d9e942b..84f3c9fd6 100644
--- a/src/main/java/org/thoughtcrime/securesms/qr/QrCodeHandler.java
+++ b/src/main/java/org/thoughtcrime/securesms/qr/QrCodeHandler.java
@@ -25,10 +25,22 @@ import org.thoughtcrime.securesms.util.views.ProgressDialog;
 import chat.delta.rpc.Rpc;
 import chat.delta.rpc.RpcException;
 
-public class QrCodeHandler {
+import chat.delta.rpc.types.SecurejoinSource;
+import chat.delta.rpc.types.SecurejoinUiPath;
 
+public class QrCodeHandler {
     private static final String TAG = QrCodeHandler.class.getSimpleName();
 
+    public static int SECUREJOIN_SOURCE_EXTERNAL_LINK = 1;
+    public static int SECUREJOIN_SOURCE_INTERNAL_LINK = 2;
+    public static int SECUREJOIN_SOURCE_CLIPBOARD = 3;
+    public static int SECUREJOIN_SOURCE_IMAGE_LOADED = 4;
+    public static int SECUREJOIN_SOURCE_SCAN = 5;
+
+    public static int SECUREJOIN_UIPATH_QR_ICON = 1;
+    public static int SECUREJOIN_UIPATH_NEW_CONTACT = 2;
+
+
     private final Activity activity;
     private final DcContext dcContext;
     private final Rpc rpc;
@@ -41,15 +53,15 @@ public class QrCodeHandler {
         accId = dcContext.getAccountId();
     }
 
-    public void onScanPerformed(IntentResult scanResult) {
+    public void onScanPerformed(IntentResult scanResult, SecurejoinUiPath uipath) {
         if (scanResult == null || scanResult.getFormatName() == null) {
             return; // aborted
         }
 
-        handleQrData(scanResult.getContents());
+        handleQrData(scanResult.getContents(), SecurejoinSource.Scan, uipath);
     }
 
-    public void handleQrData(String rawString) {
+    public void handleQrData(String rawString, SecurejoinSource source, SecurejoinUiPath uiPath) {
         AlertDialog.Builder builder = new AlertDialog.Builder(activity);
         final DcLot qrParsed = dcContext.checkQr(rawString);
         String name = dcContext.getContact(qrParsed.getId()).getDisplayName();
@@ -57,7 +69,7 @@ public class QrCodeHandler {
             case DcContext.DC_QR_ASK_VERIFYCONTACT:
             case DcContext.DC_QR_ASK_VERIFYGROUP:
             case DcContext.DC_QR_ASK_JOIN_BROADCAST:
-                showVerifyContactOrGroup(activity, builder, rawString, qrParsed, name);
+                showVerifyContactOrGroup(activity, builder, rawString, qrParsed, name, source, uiPath);
                 break;
 
             case DcContext.DC_QR_FPR_WITHOUT_ADDR:
@@ -224,7 +236,13 @@ public class QrCodeHandler {
         });
     }
 
-    private void showVerifyContactOrGroup(Activity activity, AlertDialog.Builder builder, String qrRawString, DcLot qrParsed, String name) {
+  private void showVerifyContactOrGroup(Activity activity,
+                                        AlertDialog.Builder builder,
+                                        String qrRawString,
+                                        DcLot qrParsed,
+                                        String name,
+                                        SecurejoinSource source,
+                                        SecurejoinUiPath uipath) {
         String msg;
         switch (qrParsed.getState()) {
             case DcContext.DC_QR_ASK_VERIFYGROUP:
@@ -239,17 +257,17 @@ public class QrCodeHandler {
         }
         builder.setMessage(msg);
         builder.setPositiveButton(android.R.string.ok, (dialogInterface, i) -> {
-            DcHelper.getEventCenter(activity).captureNextError();
-            int newChatId = dcContext.joinSecurejoin(qrRawString);
-            DcHelper.getEventCenter(activity).endCaptureNextError();
+            try {
+                int newChatId = DcHelper.getRpc(activity).secureJoinWithUxInfo(dcContext.getAccountId(), qrRawString, source, uipath);
+                if (newChatId == 0) throw new Exception("Securejoin failed to create a chat");
 
-            if (newChatId != 0) {
                 Intent intent = new Intent(activity, ConversationActivity.class);
                 intent.putExtra(ConversationActivity.CHAT_ID_EXTRA, newChatId);
                 activity.startActivity(intent);
-            } else {
+            } catch (Exception e) {
+                e.printStackTrace();
                 AlertDialog.Builder builder1 = new AlertDialog.Builder(activity);
-                builder1.setMessage(dcContext.getLastError());
+                builder1.setMessage(e.getMessage());
                 builder1.setPositiveButton(android.R.string.ok, null);
                 builder1.create().show();
             }
diff --git a/src/main/java/org/thoughtcrime/securesms/relay/RelayListActivity.java b/src/main/java/org/thoughtcrime/securesms/relay/RelayListActivity.java
index d0ae1859d..bb15e4dfc 100644
--- a/src/main/java/org/thoughtcrime/securesms/relay/RelayListActivity.java
+++ b/src/main/java/org/thoughtcrime/securesms/relay/RelayListActivity.java
@@ -32,6 +32,8 @@ import java.util.List;
 import chat.delta.rpc.Rpc;
 import chat.delta.rpc.RpcException;
 import chat.delta.rpc.types.EnteredLoginParam;
+import chat.delta.rpc.types.SecurejoinSource;
+import chat.delta.rpc.types.SecurejoinUiPath;
 
 public class RelayListActivity extends BaseActionBarActivity
   implements RelayListAdapter.OnRelayClickListener, DcEventCenter.DcEventDelegate {
@@ -88,7 +90,7 @@ public class RelayListActivity extends BaseActionBarActivity
     String qrdata = getIntent().getStringExtra(EXTRA_QR_DATA);
     if (qrdata != null) {
       QrCodeHandler qrCodeHandler = new QrCodeHandler(this);
-      qrCodeHandler.handleQrData(qrdata);
+      qrCodeHandler.handleQrData(qrdata, SecurejoinSource.Unknown, SecurejoinUiPath.Unknown);
     }
   }
 
@@ -173,7 +175,7 @@ public class RelayListActivity extends BaseActionBarActivity
     if (requestCode == IntentIntegrator.REQUEST_CODE) {
       IntentResult scanResult = IntentIntegrator.parseActivityResult(requestCode, resultCode, data);
       QrCodeHandler qrCodeHandler = new QrCodeHandler(this);
-      qrCodeHandler.onScanPerformed(scanResult);
+      qrCodeHandler.onScanPerformed(scanResult, SecurejoinUiPath.Unknown);
     }
   }
 
diff --git a/src/main/java/org/thoughtcrime/securesms/util/LongClickCopySpan.java b/src/main/java/org/thoughtcrime/securesms/util/LongClickCopySpan.java
index 7f64ca54f..606bedbeb 100644
--- a/src/main/java/org/thoughtcrime/securesms/util/LongClickCopySpan.java
+++ b/src/main/java/org/thoughtcrime/securesms/util/LongClickCopySpan.java
@@ -19,6 +19,8 @@ import org.thoughtcrime.securesms.R;
 import org.thoughtcrime.securesms.connect.DcHelper;
 import org.thoughtcrime.securesms.qr.QrCodeHandler;
 
+import chat.delta.rpc.types.SecurejoinSource;
+
 public class LongClickCopySpan extends ClickableSpan {
   private static final String PREFIX_MAILTO = "mailto:";
   private static final String PREFIX_TEL = "tel:";
@@ -80,13 +82,13 @@ public class LongClickCopySpan extends ClickableSpan {
       }
     } else if (Util.isInviteURL(url)) {
       QrCodeHandler qrCodeHandler = new QrCodeHandler((Activity) widget.getContext());
-      qrCodeHandler.handleQrData(url);
+      qrCodeHandler.handleQrData(url, SecurejoinSource.InternalLink, null);
     } else {
       Activity activity = (Activity) widget.getContext();
       DcContext dcContext = DcHelper.getContext(activity);
       if (dcContext.checkQr(url).getState() == DcContext.DC_QR_PROXY) {
         QrCodeHandler qrCodeHandler = new QrCodeHandler(activity);
-        qrCodeHandler.handleQrData(url);
+        qrCodeHandler.handleQrData(url, null, null);
       } else {
         IntentUtils.showInBrowser(widget.getContext(), url);
       }
diff --git a/src/main/java/org/thoughtcrime/securesms/util/Prefs.java b/src/main/java/org/thoughtcrime/securesms/util/Prefs.java
index 4e257d843..52d0d2e3e 100644
--- a/src/main/java/org/thoughtcrime/securesms/util/Prefs.java
+++ b/src/main/java/org/thoughtcrime/securesms/util/Prefs.java
@@ -12,12 +12,10 @@ import androidx.annotation.NonNull;
 import androidx.annotation.Nullable;
 import androidx.core.app.NotificationCompat;
 
-import com.b44t.messenger.DcAccounts;
 import com.b44t.messenger.DcContext;
 
 import org.thoughtcrime.securesms.BuildConfig;
 import org.thoughtcrime.securesms.connect.DcHelper;
-import org.thoughtcrime.securesms.notifications.FcmReceiveService;
 import org.thoughtcrime.securesms.preferences.widgets.NotificationPrivacyPreference;
 
 import java.util.ArrayList;
@@ -46,6 +44,7 @@ public class Prefs {
   public  static final String SCREEN_SECURITY_PREF             = "pref_screen_security";
   private static final String ENTER_SENDS_PREF                 = "pref_enter_sends";
   private static final String PROMPTED_DOZE_MSG_ID_PREF        = "pref_prompted_doze_msg_id";
+  private static final String STATS_DEVICE_MSG_ID_PREF         = "pref_stats_device_msg_id";
   public  static final String DOZE_ASKED_DIRECTLY              = "pref_doze_asked_directly";
   public  static final String ASKED_FOR_NOTIFICATION_PERMISSION= "pref_asked_for_notification_permission";
   private static final String IN_THREAD_NOTIFICATION_PREF      = "pref_key_inthread_notifications";
@@ -154,6 +153,14 @@ public class Prefs {
     return getIntegerPreference(context, PROMPTED_DOZE_MSG_ID_PREF, 0);
   }
 
+  public static void setStatsDeviceMsgId(Context context, int msg_id) {
+    setIntegerPreference(context, STATS_DEVICE_MSG_ID_PREF, msg_id);
+  }
+
+  public static int getStatsDeviceMsgId(Context context) {
+    return getIntegerPreference(context, STATS_DEVICE_MSG_ID_PREF, 0);
+  }
+
   public static boolean isPushEnabled(Context context) {
       return BuildConfig.USE_PLAY_SERVICES;
   }
diff --git a/src/main/res/values/strings.xml b/src/main/res/values/strings.xml
index bd0a5ad77..d4251d01c 100644
--- a/src/main/res/values/strings.xml
+++ b/src/main/res/values/strings.xml
@@ -824,8 +824,13 @@
     <string name="disable_imap_idle">Disable IMAP IDLE</string>
     <string name="disable_imap_idle_explain">Do not use IMAP IDLE extension even if the server supports it. Enabling this option will delay message retrieval, enable it only for testing.</string>
     <string name="send_stats_to_devs">Send statistics to Delta Chat\'s developers</string>
-    <string name="stats_msg_body">The attachment contains anonymous usage statistics, which help us improve Delta Chat. Thank you!</string>
-
+    <string name="stats_device_message">Do you want to help improve Delta Chat and support research by sending weekly anonymous usage statistics?\n\n👉 Tap here… 👈</string>
+    <string name="stats_confirmation_dialog">Do you want to help improve Delta Chat and support research by sending weekly anonymous usage statistics?</string>
+    <string name="stats_thanks">Thank you! You can always disable the sending at \"Settings -> Advanced\".\n\nDo you additionally have 5 minutes to take part in a scientific study on the security of Delta Chat?</string>
+    <string name="stats_disable_dialog">Sending of statistics is already enabled.\n\nDo you want to disable it?</string>
+    <string name="disable">Disable</string>
+    <string name="stats_keep_sending">Keep sending</string>
+    <string name="stats_msg_body">The attachment contains anonymous usage statistics, which helps us improve Delta Chat. See https://delta.chat/help#statssending for more information. Thank you!</string>
 
     <!-- Emoji picker and categories -->
     <string name="emoji_search_results">Search Results</string>
diff --git a/src/main/res/xml/preferences_advanced.xml b/src/main/res/xml/preferences_advanced.xml
index dc540cb4f..dde838992 100644
--- a/src/main/res/xml/preferences_advanced.xml
+++ b/src/main/res/xml/preferences_advanced.xml
@@ -49,6 +49,10 @@
     </PreferenceCategory>
 
     <PreferenceCategory>
+        <org.thoughtcrime.securesms.components.SwitchPreferenceCompat
+            android:defaultValue="false"
+            android:key="pref_stats_sending"
+            android:title="@string/send_stats_to_devs" />
 
         <org.thoughtcrime.securesms.components.SwitchPreferenceCompat
             android:defaultValue="false"
```

## Chatmail Core

I implemented the core logic of statistics-sending in https://github.com/chatmail/core/. The diff of my contribution is:

```diff
diff --git a/assets/self-reporting-bot.vcf b/assets/statistics-bot.vcf
similarity index 100%
rename from assets/self-reporting-bot.vcf
rename to assets/statistics-bot.vcf
diff --git a/deltachat-jsonrpc/src/api.rs b/deltachat-jsonrpc/src/api.rs
index 595f6e989..5171e2a14 100644
--- a/deltachat-jsonrpc/src/api.rs
+++ b/deltachat-jsonrpc/src/api.rs
@@ -66,7 +66,7 @@
     },
 };
 use crate::api::types::chat_list::{get_chat_list_item_by_id, ChatListItemFetchResult};
-use crate::api::types::qr::QrObject;
+use crate::api::types::qr::{QrObject, SecurejoinSource, SecurejoinUiPath};
 
 #[derive(Debug)]
 struct AccountState {
@@ -381,11 +381,6 @@ async fn copy_to_blob_dir(&self, account_id: u32, path: String) -> Result<PathBu
         Ok(BlobObject::create_and_deduplicate(&ctx, file, file)?.to_abs_path())
     }
 
-    async fn draft_self_report(&self, account_id: u32) -> Result<u32> {
-        let ctx = self.get_context(account_id).await?;
-        Ok(ctx.draft_self_report().await?.to_u32())
-    }
-
     /// Sets the given configuration key.
     async fn set_config(&self, account_id: u32, key: String, value: Option<String>) -> Result<()> {
         let ctx = self.get_context(account_id).await?;
@@ -884,6 +879,38 @@ async fn secure_join(&self, account_id: u32, qr: String) -> Result<u32> {
         Ok(chat_id.to_u32())
     }
 
+    /// Like `secure_join()`, but allows to pass a source and a UI-path.
+    /// You only need this if your UI has an option to send statistics
+    /// to Delta Chat's developers.
+    ///
+    /// **source**: The source where the QR code came from.
+    /// E.g. a link that was clicked inside or outside Delta Chat,
+    /// the "Paste from Clipboard" action,
+    /// the "Load QR code as image" action,
+    /// or a QR code scan.
+    ///
+    /// **uipath**: Which UI path did the user use to arrive at the QR code screen.
+    /// If the SecurejoinSource was ExternalLink or InternalLink,
+    /// pass `None` here, because the QR code screen wasn't even opened.
+    /// ```
+    async fn secure_join_with_ux_info(
+        &self,
+        account_id: u32,
+        qr: String,
+        source: Option<SecurejoinSource>,
+        uipath: Option<SecurejoinUiPath>,
+    ) -> Result<u32> {
+        let ctx = self.get_context(account_id).await?;
+        let chat_id = securejoin::join_securejoin_with_ux_info(
+            &ctx,
+            &qr,
+            source.map(Into::into),
+            uipath.map(Into::into),
+        )
+        .await?;
+        Ok(chat_id.to_u32())
+    }
+
     async fn leave_group(&self, account_id: u32, chat_id: u32) -> Result<()> {
         let ctx = self.get_context(account_id).await?;
         remove_contact_from_chat(&ctx, ChatId::new(chat_id), ContactId::SELF).await
diff --git a/deltachat-jsonrpc/src/api/types/qr.rs b/deltachat-jsonrpc/src/api/types/qr.rs
index 0414ec9e5..8270e0da6 100644
--- a/deltachat-jsonrpc/src/api/types/qr.rs
+++ b/deltachat-jsonrpc/src/api/types/qr.rs
@@ -1,4 +1,5 @@
 use deltachat::qr::Qr;
+use serde::Deserialize;
 use serde::Serialize;
 use typescript_type_def::TypeDef;
 
@@ -304,3 +305,53 @@ fn from(qr: Qr) -> Self {
         }
     }
 }
+
+#[derive(Deserialize, TypeDef, schemars::JsonSchema)]
+pub enum SecurejoinSource {
+    /// Because of some problem, it is unknown where the QR code came from.
+    Unknown,
+    /// The user opened a link somewhere outside Delta Chat
+    ExternalLink,
+    /// The user clicked on a link in a message inside Delta Chat
+    InternalLink,
+    /// The user clicked "Paste from Clipboard" in the QR scan activity
+    Clipboard,
+    /// The user clicked "Load QR code as image" in the QR scan activity
+    ImageLoaded,
+    /// The user scanned a QR code
+    Scan,
+}
+
+#[derive(Deserialize, TypeDef, schemars::JsonSchema)]
+pub enum SecurejoinUiPath {
+    /// The UI path is unknown, or the user didn't open the QR code screen at all.
+    Unknown,
+    /// The user directly clicked on the QR icon in the main screen
+    QrIcon,
+    /// The user first clicked on the `+` button in the main screen,
+    /// and then on "New Contact"
+    NewContact,
+}
+
+impl From<SecurejoinSource> for deltachat::SecurejoinSource {
+    fn from(value: SecurejoinSource) -> Self {
+        match value {
+            SecurejoinSource::Unknown => deltachat::SecurejoinSource::Unknown,
+            SecurejoinSource::ExternalLink => deltachat::SecurejoinSource::ExternalLink,
+            SecurejoinSource::InternalLink => deltachat::SecurejoinSource::InternalLink,
+            SecurejoinSource::Clipboard => deltachat::SecurejoinSource::Clipboard,
+            SecurejoinSource::ImageLoaded => deltachat::SecurejoinSource::ImageLoaded,
+            SecurejoinSource::Scan => deltachat::SecurejoinSource::Scan,
+        }
+    }
+}
+
+impl From<SecurejoinUiPath> for deltachat::SecurejoinUiPath {
+    fn from(value: SecurejoinUiPath) -> Self {
+        match value {
+            SecurejoinUiPath::Unknown => deltachat::SecurejoinUiPath::Unknown,
+            SecurejoinUiPath::QrIcon => deltachat::SecurejoinUiPath::QrIcon,
+            SecurejoinUiPath::NewContact => deltachat::SecurejoinUiPath::NewContact,
+        }
+    }
+}
diff --git a/deltachat-time/src/lib.rs b/deltachat-time/src/lib.rs
index c8d7d6f6c..b3b8b2a21 100644
--- a/deltachat-time/src/lib.rs
+++ b/deltachat-time/src/lib.rs
@@ -20,6 +20,11 @@ pub fn now() -> SystemTime {
     pub fn shift(duration: Duration) {
         *SYSTEM_TIME_SHIFT.write().unwrap() += duration;
     }
+
+    /// Simulates the system clock being rewound by `duration`.
+    pub fn shift_back(duration: Duration) {
+        *SYSTEM_TIME_SHIFT.write().unwrap() -= duration;
+    }
 }
 
 #[cfg(test)]
diff --git a/src/config.rs b/src/config.rs
index 1f73262b2..aabec2455 100644
--- a/src/config.rs
+++ b/src/config.rs
@@ -14,7 +14,6 @@
 
 use crate::blob::BlobObject;
 use crate::configure::EnteredLoginParam;
-use crate::constants;
 use crate::context::Context;
 use crate::events::EventType;
 use crate::log::{LogExt, info};
@@ -23,6 +22,7 @@
 use crate::provider::{Provider, get_provider_by_id};
 use crate::sync::{self, Sync::*, SyncData};
 use crate::tools::get_abs_path;
+use crate::{constants, stats};
 
 /// The available configuration keys.
 #[derive(
@@ -408,9 +408,22 @@ pub enum Config {
     /// used for signatures, encryption to self and included in `Autocrypt` header.
     KeyId,
 
-    /// This key is sent to the self_reporting bot so that the bot can recognize the user
+    /// Send statistics to Delta Chat's developers.
+    /// Can be exposed to the user as a setting.
+    StatsSending,
+
+    /// Last time statistics were sent to Delta Chat's developers
+    StatsLastSent,
+
+    /// Last time `update_message_stats()` was called
+    StatsLastUpdate,
+
+    /// This key is sent to the statistics bot so that the bot can recognize the user
     /// without storing the email address
-    SelfReportingId,
+    StatsId,
+
+    /// The last contact id that already existed when statistics-sending was enabled for the first time.
+    StatsLastOldContactId,
 
     /// MsgId of webxdc map integration.
     WebxdcIntegration,
@@ -576,8 +589,9 @@ pub(crate) async fn get_config_bool_opt(&self, key: Config) -> Result<Option<boo
     /// Returns boolean configuration value for the given key.
     pub async fn get_config_bool(&self, key: Config) -> Result<bool> {
         Ok(self
-            .get_config_parsed::<i32>(key)
+            .get_config(key)
             .await?
+            .and_then(|s| s.parse::<i32>().ok())
             .map(|x| x != 0)
             .unwrap_or_default())
     }
@@ -716,10 +730,19 @@ pub async fn set_config(&self, key: Config, value: Option<&str>) -> Result<()> {
             true => self.scheduler.pause(self).await?,
             _ => Default::default(),
         };
+        if key == Config::StatsSending {
+            let old_value = self.get_config(key).await?;
+            let old_value = bool_from_config(old_value.as_deref());
+            let new_value = bool_from_config(value);
+            stats::pre_sending_config_change(self, old_value, new_value).await?;
+        }
         self.set_config_internal(key, value).await?;
         if key == Config::SentboxWatch {
             self.last_full_folder_scan.lock().await.take();
         }
+        if key == Config::StatsSending {
+            stats::maybe_send_stats(self).await?;
+        }
         Ok(())
     }
 
@@ -871,6 +894,10 @@ pub(crate) fn from_bool(val: bool) -> Option<&'static str> {
     Some(if val { "1" } else { "0" })
 }
 
+pub(crate) fn bool_from_config(config: Option<&str>) -> bool {
+    config.is_some_and(|v| v.parse::<i32>().unwrap_or_default() != 0)
+}
+
 // Separate impl block for self address handling
 impl Context {
     /// Determine whether the specified addr maps to the/a self addr.
diff --git a/src/constants.rs b/src/constants.rs
index b1275fd21..b536d040c 100644
--- a/src/constants.rs
+++ b/src/constants.rs
@@ -101,6 +101,8 @@ pub enum MediaQuality {
     Copy,
     PartialEq,
     Eq,
+    PartialOrd,
+    Ord,
     FromPrimitive,
     ToPrimitive,
     FromSql,
diff --git a/src/context.rs b/src/context.rs
index 8d160ae9f..9dd664836 100644
--- a/src/context.rs
+++ b/src/context.rs
@@ -10,28 +10,22 @@
 
 use anyhow::{Context as _, Result, bail, ensure};
 use async_channel::{self as channel, Receiver, Sender};
-use pgp::types::PublicKeyTrait;
 use ratelimit::Ratelimit;
 use tokio::sync::{Mutex, Notify, RwLock};
 
 use crate::chat::{ChatId, get_chat_cnt};
-use crate::chatlist_events;
 use crate::config::Config;
-use crate::constants::{
-    self, DC_BACKGROUND_FETCH_QUOTA_CHECK_RATELIMIT, DC_CHAT_ID_TRASH, DC_VERSION_STR,
-};
-use crate::contact::{Contact, ContactId, import_vcard, mark_contact_id_as_verified};
+use crate::constants::{self, DC_BACKGROUND_FETCH_QUOTA_CHECK_RATELIMIT, DC_VERSION_STR};
+use crate::contact::{Contact, ContactId};
 use crate::debug_logging::DebugLogging;
-use crate::download::DownloadState;
 use crate::events::{Event, EventEmitter, EventType, Events};
 use crate::imap::{FolderMeaning, Imap, ServerMetadata};
-use crate::key::{load_self_secret_key, self_fingerprint};
+use crate::key::self_fingerprint;
 use crate::log::{info, warn};
 use crate::logged_debug_assert;
 use crate::login_param::{ConfiguredLoginParam, EnteredLoginParam};
-use crate::message::{self, Message, MessageState, MsgId};
+use crate::message::{self, MessageState, MsgId};
 use crate::net::tls::TlsSessionStore;
-use crate::param::{Param, Params};
 use crate::peer_channels::Iroh;
 use crate::push::PushSubscriber;
 use crate::quota::QuotaInfo;
@@ -39,7 +33,8 @@
 use crate::sql::Sql;
 use crate::stock_str::StockStrings;
 use crate::timesmearing::SmearedTimestamp;
-use crate::tools::{self, create_id, duration_to_str, time, time_elapsed};
+use crate::tools::{self, duration_to_str, time, time_elapsed};
+use crate::{chatlist_events, stats};
 
 /// Builder for the [`Context`].
 ///
@@ -1066,6 +1061,22 @@ pub async fn get_info(&self) -> Result<BTreeMap<&'static str, String>> {
                 .await?
                 .unwrap_or_default(),
         );
+        res.insert(
+            "stats_id",
+            self.get_config(Config::StatsId)
+                .await?
+                .unwrap_or_else(|| "<unset>".to_string()),
+        );
+        res.insert(
+            "stats_sending",
+            stats::should_send_stats(self).await?.to_string(),
+        );
+        res.insert(
+            "stats_last_sent",
+            self.get_config_i64(Config::StatsLastSent)
+                .await?
+                .to_string(),
+        );
         res.insert(
             "fail_on_receiving_full_msg",
             self.sql
@@ -1080,138 +1091,6 @@ pub async fn get_info(&self) -> Result<BTreeMap<&'static str, String>> {
         Ok(res)
     }
 
-    async fn get_self_report(&self) -> Result<String> {
-        #[derive(Default)]
-        struct ChatNumbers {
-            opportunistic_dc: u32,
-            opportunistic_mua: u32,
-            unencrypted_dc: u32,
-            unencrypted_mua: u32,
-        }
-
-        let mut res = String::new();
-        res += &format!("core_version {}\n", get_version_str());
-
-        let num_msgs: u32 = self
-            .sql
-            .query_get_value(
-                "SELECT COUNT(*) FROM msgs WHERE hidden=0 AND chat_id!=?",
-                (DC_CHAT_ID_TRASH,),
-            )
-            .await?
-            .unwrap_or_default();
-        res += &format!("num_msgs {num_msgs}\n");
-
-        let num_chats: u32 = self
-            .sql
-            .query_get_value("SELECT COUNT(*) FROM chats WHERE id>9 AND blocked!=1", ())
-            .await?
-            .unwrap_or_default();
-        res += &format!("num_chats {num_chats}\n");
-
-        let db_size = tokio::fs::metadata(&self.sql.dbfile).await?.len();
-        res += &format!("db_size_bytes {db_size}\n");
-
-        let secret_key = &load_self_secret_key(self).await?.primary_key;
-        let key_created = secret_key.public_key().created_at().timestamp();
-        res += &format!("key_created {key_created}\n");
-
-        // how many of the chats active in the last months are:
-        // - opportunistic-encrypted and the contact uses Delta Chat
-        // - opportunistic-encrypted and the contact uses a classical MUA
-        // - unencrypted and the contact uses Delta Chat
-        // - unencrypted and the contact uses a classical MUA
-        let three_months_ago = time().saturating_sub(3600 * 24 * 30 * 3);
-        let chats = self
-            .sql
-            .query_map(
-                "SELECT m.param, m.msgrmsg
-                    FROM chats c
-                    JOIN msgs m
-                        ON c.id=m.chat_id
-                        AND m.id=(
-                                SELECT id
-                                FROM msgs
-                                WHERE chat_id=c.id
-                                AND hidden=0
-                                AND download_state=?
-                                AND to_id!=?
-                                ORDER BY timestamp DESC, id DESC LIMIT 1)
-                    WHERE c.id>9
-                    AND (c.blocked=0 OR c.blocked=2)
-                    AND IFNULL(m.timestamp,c.created_timestamp) > ?
-                    GROUP BY c.id",
-                (DownloadState::Done, ContactId::INFO, three_months_ago),
-                |row| {
-                    let message_param: Params =
-                        row.get::<_, String>(1)?.parse().unwrap_or_default();
-                    let is_dc_message: bool = row.get(2)?;
-                    Ok((message_param, is_dc_message))
-                },
-                |rows| {
-                    let mut chats = ChatNumbers::default();
-                    for row in rows {
-                        let (message_param, is_dc_message) = row?;
-                        let encrypted = message_param
-                            .get_bool(Param::GuaranteeE2ee)
-                            .unwrap_or(false);
-
-                        if encrypted {
-                            if is_dc_message {
-                                chats.opportunistic_dc += 1;
-                            } else {
-                                chats.opportunistic_mua += 1;
-                            }
-                        } else if is_dc_message {
-                            chats.unencrypted_dc += 1;
-                        } else {
-                            chats.unencrypted_mua += 1;
-                        }
-                    }
-                    Ok(chats)
-                },
-            )
-            .await?;
-        res += &format!("chats_opportunistic_dc {}\n", chats.opportunistic_dc);
-        res += &format!("chats_opportunistic_mua {}\n", chats.opportunistic_mua);
-        res += &format!("chats_unencrypted_dc {}\n", chats.unencrypted_dc);
-        res += &format!("chats_unencrypted_mua {}\n", chats.unencrypted_mua);
-
-        let self_reporting_id = match self.get_config(Config::SelfReportingId).await? {
-            Some(id) => id,
-            None => {
-                let id = create_id();
-                self.set_config(Config::SelfReportingId, Some(&id)).await?;
-                id
-            }
-        };
-        res += &format!("self_reporting_id {self_reporting_id}");
-
-        Ok(res)
-    }
-
-    /// Drafts a message with statistics about the usage of Delta Chat.
-    /// The user can inspect the message if they want, and then hit "Send".
-    ///
-    /// On the other end, a bot will receive the message and make it available
-    /// to Delta Chat's developers.
-    pub async fn draft_self_report(&self) -> Result<ChatId> {
-        const SELF_REPORTING_BOT_VCARD: &str = include_str!("../assets/self-reporting-bot.vcf");
-        let contact_id: ContactId = *import_vcard(self, SELF_REPORTING_BOT_VCARD)
-            .await?
-            .first()
-            .context("Self reporting bot vCard does not contain a contact")?;
-        mark_contact_id_as_verified(self, contact_id, Some(ContactId::SELF)).await?;
-
-        let chat_id = ChatId::create_for_contact(self, contact_id).await?;
-
-        let mut msg = Message::new_text(self.get_self_report().await?);
-
-        chat_id.set_draft(self, Some(&mut msg)).await?;
-
-        Ok(chat_id)
-    }
-
     /// Get a list of fresh, unmuted messages in unblocked chats.
     ///
     /// The list starts with the most recent message
diff --git a/src/context/context_tests.rs b/src/context/context_tests.rs
index 03dcaf3bb..4a20c3af3 100644
--- a/src/context/context_tests.rs
+++ b/src/context/context_tests.rs
@@ -6,9 +6,9 @@
 use crate::chat::{Chat, MuteDuration, get_chat_contacts, get_chat_msgs, send_msg, set_muted};
 use crate::chatlist::Chatlist;
 use crate::constants::Chattype;
-use crate::mimeparser::SystemMessage;
+use crate::message::Message;
 use crate::receive_imf::receive_imf;
-use crate::test_utils::{E2EE_INFO_MSGS, TestContext, get_chat_msg};
+use crate::test_utils::{E2EE_INFO_MSGS, TestContext};
 use crate::tools::{SystemTime, create_outgoing_rfc724_mid};
 
 #[tokio::test(flavor = "multi_thread", worker_threads = 2)]
@@ -276,7 +276,6 @@ async fn test_get_info_completeness() {
         "mail_port",
         "mail_security",
         "notify_about_wrong_pw",
-        "self_reporting_id",
         "selfstatus",
         "send_server",
         "send_user",
@@ -296,6 +295,8 @@ async fn test_get_info_completeness() {
         "webxdc_integration",
         "device_token",
         "encrypted_device_token",
+        "stats_last_update",
+        "stats_last_old_contact_id",
     ];
     let t = TestContext::new().await;
     let info = t.get_info().await.unwrap();
@@ -598,23 +599,6 @@ async fn test_get_next_msgs() -> Result<()> {
     Ok(())
 }
 
-#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
-async fn test_draft_self_report() -> Result<()> {
-    let alice = TestContext::new_alice().await;
-
-    let chat_id = alice.draft_self_report().await?;
-    let msg = get_chat_msg(&alice, chat_id, 0, 1).await;
-    assert_eq!(msg.get_info_type(), SystemMessage::ChatE2ee);
-
-    let mut draft = chat_id.get_draft(&alice).await?.unwrap();
-    assert!(draft.text.starts_with("core_version"));
-
-    // Test that sending into the chat works:
-    let _sent = alice.send_msg(chat_id, &mut draft).await;
-
-    Ok(())
-}
-
 #[tokio::test(flavor = "multi_thread", worker_threads = 2)]
 async fn test_cache_is_cleared_when_io_is_started() -> Result<()> {
     let alice = TestContext::new_alice().await;
diff --git a/src/ephemeral.rs b/src/ephemeral.rs
index 84a580517..c7a56774a 100644
--- a/src/ephemeral.rs
+++ b/src/ephemeral.rs
@@ -80,12 +80,12 @@
 use crate::context::Context;
 use crate::download::MIN_DELETE_SERVER_AFTER;
 use crate::events::EventType;
-use crate::location;
 use crate::log::{LogExt, error, info, warn};
 use crate::message::{Message, MessageState, MsgId, Viewtype};
 use crate::mimeparser::SystemMessage;
 use crate::stock_str;
 use crate::tools::{SystemTime, duration_to_str, time};
+use crate::{location, stats};
 
 /// Ephemeral timer value.
 #[derive(Debug, PartialEq, Eq, Copy, Clone, Serialize, Deserialize)]
@@ -610,7 +610,7 @@ pub(crate) async fn ephemeral_loop(context: &Context, interrupt_receiver: Receiv
                 + Duration::from_secs(1)
         } else {
             // no messages to be deleted for now, wait long for one to occur
-            now + Duration::from_secs(86400)
+            now + Duration::from_secs(86400) // 1 day
         };
 
         if let Ok(duration) = until.duration_since(now) {
@@ -637,6 +637,12 @@ pub(crate) async fn ephemeral_loop(context: &Context, interrupt_receiver: Receiv
             }
         }
 
+        // Make sure that the statistics stay correct by updating them _before_ deleting messages:
+        stats::maybe_update_message_stats(context)
+            .await
+            .log_err(context)
+            .ok();
+
         delete_expired_messages(context, time())
             .await
             .log_err(context)
diff --git a/src/lib.rs b/src/lib.rs
index 6a33b23c9..d3d30e386 100644
--- a/src/lib.rs
+++ b/src/lib.rs
@@ -99,6 +99,9 @@
 pub mod net;
 pub mod plaintext;
 mod push;
+mod stats;
+pub use stats::SecurejoinSource;
+pub use stats::SecurejoinUiPath;
 pub mod summary;
 
 mod debug_logging;
diff --git a/src/receive_imf.rs b/src/receive_imf.rs
index e64a74a7e..11e74ab63 100644
--- a/src/receive_imf.rs
+++ b/src/receive_imf.rs
@@ -40,6 +40,7 @@
 use crate::rusqlite::OptionalExtension;
 use crate::securejoin::{self, handle_securejoin_handshake, observe_securejoin_on_other_device};
 use crate::simplify;
+use crate::stats::STATISTICS_BOT_EMAIL;
 use crate::stock_str;
 use crate::sync::Sync::*;
 use crate::tools::{self, buf_compress, remove_subject_prefix};
@@ -1700,6 +1701,8 @@ async fn add_parts(
     // No check for `hidden` because only reactions are such and they should be `InFresh`.
     {
         MessageState::InSeen
+    } else if mime_parser.from.addr == STATISTICS_BOT_EMAIL {
+        MessageState::InNoticed
     } else {
         MessageState::InFresh
     };
diff --git a/src/scheduler.rs b/src/scheduler.rs
index bbf1ef016..0c1134e43 100644
--- a/src/scheduler.rs
+++ b/src/scheduler.rs
@@ -15,7 +15,6 @@
 
 pub(crate) use self::connectivity::ConnectivityStore;
 use crate::config::{self, Config};
-use crate::constants;
 use crate::contact::{ContactId, RecentlySeenLoop};
 use crate::context::Context;
 use crate::download::{DownloadState, download_msg};
@@ -27,7 +26,9 @@
 use crate::message::MsgId;
 use crate::smtp::{Smtp, send_smtp_messages};
 use crate::sql;
+use crate::stats::maybe_send_stats;
 use crate::tools::{self, duration_to_str, maybe_add_time_based_warnings, time, time_elapsed};
+use crate::{constants, stats};
 
 pub(crate) mod connectivity;
 
@@ -513,6 +514,7 @@ async fn inbox_fetch_idle(ctx: &Context, imap: &mut Imap, mut session: Session)
         }
     };
 
+    maybe_send_stats(ctx).await.log_err(ctx).ok();
     match ctx.get_config_bool(Config::FetchedExistingMsgs).await {
         Ok(fetched_existing_msgs) => {
             if !fetched_existing_msgs {
@@ -807,6 +809,11 @@ async fn smtp_loop(
                 }
             }
 
+            stats::maybe_update_message_stats(&ctx)
+                .await
+                .log_err(&ctx)
+                .ok();
+
             // Fake Idle
             info!(ctx, "SMTP fake idle started.");
             match &connection.last_send_error {
diff --git a/src/securejoin.rs b/src/securejoin.rs
index 4e3bea5bb..92cfd8998 100644
--- a/src/securejoin.rs
+++ b/src/securejoin.rs
@@ -14,6 +14,7 @@
 use crate::events::EventType;
 use crate::headerdef::HeaderDef;
 use crate::key::{DcKey, Fingerprint, load_self_public_key};
+use crate::log::LogExt as _;
 use crate::log::{error, info, warn};
 use crate::message::{Message, Viewtype};
 use crate::mimeparser::{MimeMessage, SystemMessage};
@@ -21,13 +22,14 @@
 use crate::qr::check_qr;
 use crate::securejoin::bob::JoinerProgress;
 use crate::sync::Sync::*;
-use crate::token;
 use crate::tools::{create_id, time};
+use crate::{SecurejoinSource, stats};
+use crate::{SecurejoinUiPath, token};
 
 mod bob;
 mod qrinvite;
 
-use qrinvite::QrInvite;
+pub(crate) use qrinvite::QrInvite;
 
 use crate::token::Namespace;
 
@@ -168,12 +170,38 @@ async fn get_self_fingerprint(context: &Context) -> Result<Fingerprint> {
 ///
 /// The function returns immediately and the handshake will run in background.
 pub async fn join_securejoin(context: &Context, qr: &str) -> Result<ChatId> {
-    securejoin(context, qr).await.map_err(|err| {
+    join_securejoin_with_ux_info(context, qr, None, None).await
+}
+
+/// Take a scanned QR-code and do the setup-contact/join-group/invite handshake.
+///
+/// This is the start of the process for the joiner.  See the module and ffi documentation
+/// for more details.
+///
+/// The function returns immediately and the handshake will run in background.
+///
+/// **source** and **uipath** are for statistics-sending,
+/// if the user enabled it in the settings;
+/// if you don't have statistics-sending implemented, just pass `None` here.
+pub async fn join_securejoin_with_ux_info(
+    context: &Context,
+    qr: &str,
+    source: Option<SecurejoinSource>,
+    uipath: Option<SecurejoinUiPath>,
+) -> Result<ChatId> {
+    let res = securejoin(context, qr).await.map_err(|err| {
         warn!(context, "Fatal joiner error: {:#}", err);
         // The user just scanned this QR code so has context on what failed.
         error!(context, "QR process failed");
         err
-    })
+    })?;
+
+    stats::count_securejoin_ux_info(context, source, uipath)
+        .await
+        .log_err(context)
+        .ok();
+
+    Ok(res)
 }
 
 async fn securejoin(context: &Context, qr: &str) -> Result<ChatId> {
@@ -187,6 +215,11 @@ async fn securejoin(context: &Context, qr: &str) -> Result<ChatId> {
 
     let invite = QrInvite::try_from(qr_scan)?;
 
+    stats::count_securejoin_invite(context, &invite)
+        .await
+        .log_err(context)
+        .ok();
+
     bob::start_protocol(context, invite).await
 }
 
diff --git a/src/sql/migrations.rs b/src/sql/migrations.rs
index 54aac6506..851fec670 100644
--- a/src/sql/migrations.rs
+++ b/src/sql/migrations.rs
@@ -1271,6 +1271,45 @@ pub async fn run(context: &Context, sql: &Sql) -> Result<(bool, bool, bool)> {
         .await?;
     }
 
+    inc_and_check(&mut migration_version, 135)?;
+    if dbversion < migration_version {
+        sql.execute_migration(
+            "CREATE TABLE stats_securejoin_sources(
+                source INTEGER PRIMARY KEY,
+                count INTEGER NOT NULL DEFAULT 0
+            ) STRICT;
+            CREATE TABLE stats_securejoin_uipaths(
+                uipath INTEGER PRIMARY KEY,
+                count INTEGER NOT NULL DEFAULT 0
+            ) STRICT;
+            CREATE TABLE stats_securejoin_invites(
+                already_existed INTEGER NOT NULL,
+                already_verified INTEGER NOT NULL,
+                type TEXT NOT NULL
+            ) STRICT;
+            CREATE TABLE stats_msgs(
+                chattype INTEGER PRIMARY KEY,
+                verified INTEGER NOT NULL DEFAULT 0,
+                unverified_encrypted INTEGER NOT NULL DEFAULT 0,
+                unencrypted INTEGER NOT NULL DEFAULT 0,
+                only_to_self INTEGER NOT NULL DEFAULT 0,
+                last_counted_msg_id INTEGER NOT NULL DEFAULT 0
+            ) STRICT;",
+            migration_version,
+        )
+        .await?;
+    }
+
+    inc_and_check(&mut migration_version, 136)?;
+    if dbversion < migration_version {
+        sql.execute_migration(
+            "CREATE TABLE stats_sending_enabled_events(timestamp INTEGER NOT NULL) STRICT;
+            CREATE TABLE stats_sending_disabled_events(timestamp INTEGER NOT NULL) STRICT;",
+            migration_version,
+        )
+        .await?;
+    }
+
     let new_version = sql
         .get_raw_config_int(VERSION_CFG)
         .await?
diff --git a/src/stats.rs b/src/stats.rs
new file mode 100644
index 000000000..d360f9681
--- /dev/null
+++ b/src/stats.rs
@@ -0,0 +1,896 @@
+//! Delta Chat has an advanced option
+//! "Send statistics to the developers of Delta Chat".
+//! If this is enabled, a JSON file with some anonymous statistics
+//! will be sent to a bot once a week.
+
+use std::collections::{BTreeMap, BTreeSet};
+
+use anyhow::{Context as _, Result};
+use deltachat_derive::FromSql;
+use num_traits::ToPrimitive;
+use pgp::types::PublicKeyTrait;
+use rusqlite::OptionalExtension;
+use serde::Serialize;
+
+use crate::chat::{self, ChatId, MuteDuration};
+use crate::config::Config;
+use crate::constants::Chattype;
+use crate::contact::{Contact, ContactId, Origin, import_vcard, mark_contact_id_as_verified};
+use crate::context::{Context, get_version_str};
+use crate::key::load_self_public_keyring;
+use crate::log::LogExt;
+use crate::message::{Message, Viewtype};
+use crate::securejoin::QrInvite;
+use crate::tools::{create_id, time};
+
+pub(crate) const STATISTICS_BOT_EMAIL: &str = "self_reporting@testrun.org";
+const STATISTICS_BOT_VCARD: &str = include_str!("../assets/statistics-bot.vcf");
+const SENDING_INTERVAL_SECONDS: i64 = 3600 * 24 * 7; // 1 week
+// const SENDING_INTERVAL_SECONDS: i64 = 60; // 1 minute (for testing)
+const MESSAGE_STATS_UPDATE_INTERVAL_SECONDS: i64 = 4 * 60; // 4 minutes (less than the lowest ephemeral messages timeout)
+
+#[derive(Serialize)]
+struct Statistics {
+    core_version: String,
+    key_create_timestamps: Vec<i64>,
+    stats_id: String,
+    is_chatmail: bool,
+    contact_stats: Vec<ContactStat>,
+    message_stats: BTreeMap<Chattype, MessageStats>,
+    securejoin_sources: SecurejoinSources,
+    securejoin_uipaths: SecurejoinUiPaths,
+    securejoin_invites: Vec<JoinedInvite>,
+    sending_enabled_timestamps: Vec<i64>,
+    sending_disabled_timestamps: Vec<i64>,
+}
+
+#[derive(Serialize, PartialEq)]
+enum VerifiedStatus {
+    Direct,
+    Transitive,
+    TransitiveViaBot,
+    Opportunistic,
+    Unencrypted,
+}
+
+#[derive(Serialize)]
+struct ContactStat {
+    #[serde(skip_serializing)]
+    id: ContactId,
+
+    verified: VerifiedStatus,
+
+    // If one of the boolean properties is false,
+    // we leave them away.
+    // This way, the Json file becomes a lot smaller.
+    #[serde(skip_serializing_if = "is_false")]
+    bot: bool,
+
+    #[serde(skip_serializing_if = "is_false")]
+    direct_chat: bool,
+
+    last_seen: u64,
+
+    #[serde(skip_serializing_if = "Option::is_none")]
+    transitive_chain: Option<u32>,
+
+    /// Whether the contact was established after stats-sending was enabled
+    #[serde(skip_serializing_if = "is_false")]
+    new: bool,
+}
+
+fn is_false(b: &bool) -> bool {
+    !b
+}
+
+#[derive(Serialize, Default)]
+struct MessageStats {
+    verified: u32,
+    unverified_encrypted: u32,
+    unencrypted: u32,
+    only_to_self: u32,
+}
+
+/// Where a securejoin invite link or QR code came from.
+/// This is only used if the user enabled StatsSending.
+#[repr(u32)]
+#[derive(
+    Debug, Clone, Copy, ToPrimitive, FromPrimitive, FromSql, PartialEq, Eq, PartialOrd, Ord,
+)]
+pub enum SecurejoinSource {
+    /// Because of some problem, it is unknown where the QR code came from.
+    Unknown = 0,
+    /// The user opened a link somewhere outside Delta Chat
+    ExternalLink = 1,
+    /// The user clicked on a link in a message inside Delta Chat
+    InternalLink = 2,
+    /// The user clicked "Paste from Clipboard" in the QR scan activity
+    Clipboard = 3,
+    /// The user clicked "Load QR code as image" in the QR scan activity
+    ImageLoaded = 4,
+    /// The user scanned a QR code
+    Scan = 5,
+}
+
+#[derive(Serialize)]
+struct SecurejoinSources {
+    unknown: u32,
+    external_link: u32,
+    internal_link: u32,
+    clipboard: u32,
+    image_loaded: u32,
+    scan: u32,
+}
+
+/// How the user opened the QR activity in order scan a QR code on Android.
+/// This is only used if the user enabled StatsSending.
+#[derive(
+    Debug, Clone, Copy, ToPrimitive, FromPrimitive, FromSql, PartialEq, Eq, PartialOrd, Ord,
+)]
+pub enum SecurejoinUiPath {
+    /// The UI path is unknown, or the user didn't open the QR code screen at all.
+    Unknown = 0,
+    /// The user directly clicked on the QR icon in the main screen
+    QrIcon = 1,
+    /// The user first clicked on the `+` button in the main screen,
+    /// and then on "New Contact"
+    NewContact = 2,
+}
+
+#[derive(Serialize)]
+struct SecurejoinUiPaths {
+    other: u32,
+    qr_icon: u32,
+    new_contact: u32,
+}
+
+/// Some information on an invite-joining event
+/// (i.e. a qr scan or a clicked link).
+#[derive(Serialize)]
+struct JoinedInvite {
+    /// Whether the contact already existed before.
+    /// If this is false, then a contact was newly created.
+    already_existed: bool,
+    /// If a contact already existed,
+    /// this tells us whether the contact was verified already.
+    already_verified: bool,
+    /// The type of the invite:
+    /// "contact" for 1:1 invites that setup a verified contact,
+    /// "group" for invites that invite to a group
+    /// and also perform the contact verification 'along the way'.
+    typ: String,
+}
+
+pub(crate) async fn pre_sending_config_change(
+    context: &Context,
+    old_value: bool,
+    new_value: bool,
+) -> Result<()> {
+    // These functions are no-ops if they were called in the past already;
+    // just call them opportunistically:
+    ensure_last_old_contact_id(context).await?;
+    // Make sure that StatsId is available for the UI,
+    // in order to open the survey with the StatsId as a parameter:
+    stats_id(context).await?;
+
+    if old_value != new_value {
+        if new_value {
+            // Only count messages sent from now on:
+            set_last_counted_msg_id(context).await?;
+        } else {
+            // Update message stats one last time in case it's enabled again in the future:
+            update_message_stats(context).await?;
+        }
+
+        let sql_table = if new_value {
+            "stats_sending_enabled_events"
+        } else {
+            "stats_sending_disabled_events"
+        };
+
+        context
+            .sql
+            .execute(&format!("INSERT INTO {sql_table} VALUES(?)"), (time(),))
+            .await?;
+    }
+
+    Ok(())
+}
+
+/// Sends a message with statistics about the usage of Delta Chat,
+/// if the last time such a message was sent
+/// was more than a week ago.
+///
+/// On the other end, a bot will receive the message and make it available
+/// to Delta Chat's developers.
+pub async fn maybe_send_stats(context: &Context) -> Result<Option<ChatId>> {
+    if should_send_stats(context).await?
+        && time_has_passed(context, Config::StatsLastSent, SENDING_INTERVAL_SECONDS).await?
+    {
+        let chat_id = send_stats(context).await?;
+
+        return Ok(Some(chat_id));
+    }
+    Ok(None)
+}
+
+pub(crate) async fn maybe_update_message_stats(context: &Context) -> Result<()> {
+    if should_send_stats(context).await?
+        && time_has_passed(
+            context,
+            Config::StatsLastUpdate,
+            MESSAGE_STATS_UPDATE_INTERVAL_SECONDS,
+        )
+        .await?
+    {
+        update_message_stats(context).await?;
+    }
+
+    Ok(())
+}
+
+async fn time_has_passed(context: &Context, config: Config, seconds: i64) -> Result<bool> {
+    let last_time = context.get_config_i64(config).await?;
+    let next_time = last_time.saturating_add(seconds);
+
+    let res = if next_time <= time() {
+        // Already set the config to the current time.
+        // This prevents infinite loops in the (unlikely) case of an error:
+        context
+            .set_config_internal(config, Some(&time().to_string()))
+            .await?;
+        true
+    } else {
+        if time() < last_time {
+            // The clock was rewound.
+            // Reset the config, so that the statistics will be sent normally in a week,
+            // or be normally updated in a few minutes.
+            context
+                .set_config_internal(config, Some(&time().to_string()))
+                .await?;
+        }
+        false
+    };
+
+    Ok(res)
+}
+
+#[allow(clippy::unused_async, unused)]
+pub(crate) async fn should_send_stats(context: &Context) -> Result<bool> {
+    #[cfg(any(target_os = "android", test))]
+    {
+        context.get_config_bool(Config::StatsSending).await
+    }
+
+    // If the user enables statistics-sending on Android,
+    // and then transfers the account to e.g. Desktop,
+    // we should not send any statistics:
+    #[cfg(not(any(target_os = "android", test)))]
+    {
+        Ok(false)
+    }
+}
+
+async fn send_stats(context: &Context) -> Result<ChatId> {
+    info!(context, "Sending statistics.");
+
+    update_message_stats(context).await?;
+
+    let chat_id = get_stats_chat_id(context).await?;
+
+    let mut msg = Message::new(Viewtype::File);
+    msg.set_text(crate::stock_str::stats_msg_body(context).await);
+
+    let stats = get_stats(context).await?;
+
+    msg.set_file_from_bytes(
+        context,
+        "statistics.txt",
+        stats.as_bytes(),
+        Some("text/plain"),
+    )?;
+
+    chat::send_msg(context, chat_id, &mut msg)
+        .await
+        .context("Failed to send statistics message")
+        .log_err(context)
+        .ok();
+
+    Ok(chat_id)
+}
+
+async fn set_last_counted_msg_id(context: &Context) -> Result<()> {
+    context
+        .sql
+        .execute(
+            "UPDATE stats_msgs
+            SET last_counted_msg_id=(SELECT MAX(id) FROM msgs)",
+            (),
+        )
+        .await?;
+
+    Ok(())
+}
+
+async fn ensure_last_old_contact_id(context: &Context) -> Result<()> {
+    if context.config_exists(Config::StatsLastOldContactId).await? {
+        // The user had statistics-sending enabled already in the past,
+        // keep the 'last old contact id' as-is
+        return Ok(());
+    }
+
+    let last_contact_id: u64 = context
+        .sql
+        .query_get_value("SELECT MAX(id) FROM contacts", ())
+        .await?
+        .unwrap_or(0);
+
+    context
+        .sql
+        .set_raw_config(
+            Config::StatsLastOldContactId.as_ref(),
+            Some(&last_contact_id.to_string()),
+        )
+        .await?;
+
+    Ok(())
+}
+
+async fn get_stats(context: &Context) -> Result<String> {
+    // The Id of the last contact that already existed when the user enabled the setting.
+    // Newer contacts will get the `new` flag set.
+    let last_old_contact = context
+        .get_config_u32(Config::StatsLastOldContactId)
+        .await?;
+
+    let key_create_timestamps: Vec<i64> = load_self_public_keyring(context)
+        .await?
+        .iter()
+        .map(|k| k.created_at().timestamp())
+        .collect();
+
+    let sending_enabled_timestamps =
+        get_timestamps(context, "stats_sending_enabled_events").await?;
+    let sending_disabled_timestamps =
+        get_timestamps(context, "stats_sending_disabled_events").await?;
+
+    let stats = Statistics {
+        core_version: get_version_str().to_string(),
+        key_create_timestamps,
+        stats_id: stats_id(context).await?,
+        is_chatmail: context.is_chatmail().await?,
+        contact_stats: get_contact_stats(context, last_old_contact).await?,
+        message_stats: get_message_stats(context).await?,
+        securejoin_sources: get_securejoin_source_stats(context).await?,
+        securejoin_uipaths: get_securejoin_uipath_stats(context).await?,
+        securejoin_invites: get_securejoin_invite_stats(context).await?,
+        sending_enabled_timestamps,
+        sending_disabled_timestamps,
+    };
+
+    Ok(serde_json::to_string_pretty(&stats)?)
+}
+
+async fn get_timestamps(context: &Context, sql_table: &str) -> Result<Vec<i64>> {
+    let res = context
+        .sql
+        .query_map(
+            &format!("SELECT timestamp FROM {sql_table} LIMIT 1000"),
+            (),
+            |row| row.get(0),
+            |rows| {
+                rows.collect::<rusqlite::Result<Vec<i64>>>()
+                    .map_err(Into::into)
+            },
+        )
+        .await?;
+
+    Ok(res)
+}
+
+pub(crate) async fn stats_id(context: &Context) -> Result<String> {
+    Ok(match context.get_config(Config::StatsId).await? {
+        Some(id) => id,
+        None => {
+            let id = create_id();
+            context
+                .set_config_internal(Config::StatsId, Some(&id))
+                .await?;
+            id
+        }
+    })
+}
+
+async fn get_stats_chat_id(context: &Context) -> Result<ChatId, anyhow::Error> {
+    let contact_id: ContactId = *import_vcard(context, STATISTICS_BOT_VCARD)
+        .await?
+        .first()
+        .context("Statistics bot vCard does not contain a contact")?;
+    mark_contact_id_as_verified(context, contact_id, Some(ContactId::SELF)).await?;
+
+    let chat_id = if let Some(res) = ChatId::lookup_by_contact(context, contact_id).await? {
+        // Already exists, no need to create.
+        res
+    } else {
+        let chat_id = ChatId::get_for_contact(context, contact_id).await?;
+        chat::set_muted(context, chat_id, MuteDuration::Forever).await?;
+        chat_id
+    };
+
+    Ok(chat_id)
+}
+
+async fn get_contact_stats(context: &Context, last_old_contact: u32) -> Result<Vec<ContactStat>> {
+    let mut verified_by_map: BTreeMap<ContactId, ContactId> = BTreeMap::new();
+    let mut bot_ids: BTreeSet<ContactId> = BTreeSet::new();
+
+    let mut contacts: Vec<ContactStat> = context
+        .sql
+        .query_map(
+            "SELECT id, fingerprint<>'', verifier, last_seen, is_bot FROM contacts c
+            WHERE id>9 AND origin>? AND addr<>?",
+            (Origin::Hidden, STATISTICS_BOT_EMAIL),
+            |row| {
+                let id = row.get(0)?;
+                let is_encrypted: bool = row.get(1)?;
+                let verifier: ContactId = row.get(2)?;
+                let last_seen: u64 = row.get(3)?;
+                let bot: bool = row.get(4)?;
+
+                let verified = match (is_encrypted, verifier) {
+                    (true, ContactId::SELF) => VerifiedStatus::Direct,
+                    (true, ContactId::UNDEFINED) => VerifiedStatus::Opportunistic,
+                    (true, _) => VerifiedStatus::Transitive, // TransitiveViaBot will be filled later
+                    (false, _) => VerifiedStatus::Unencrypted,
+                };
+
+                if verifier != ContactId::UNDEFINED {
+                    verified_by_map.insert(id, verifier);
+                }
+
+                if bot {
+                    bot_ids.insert(id);
+                }
+
+                Ok(ContactStat {
+                    id,
+                    verified,
+                    bot,
+                    direct_chat: false, // will be filled later
+                    last_seen,
+                    transitive_chain: None, // will be filled later
+                    new: id.to_u32() > last_old_contact,
+                })
+            },
+            |rows| {
+                rows.collect::<std::result::Result<Vec<_>, _>>()
+                    .map_err(Into::into)
+            },
+        )
+        .await?;
+
+    // Fill TransitiveViaBot and transitive_chain
+    for contact in &mut contacts {
+        if contact.verified == VerifiedStatus::Transitive {
+            let mut transitive_chain: u32 = 0;
+            let mut has_bot = false;
+            let mut current_verifier_id = contact.id;
+
+            while current_verifier_id != ContactId::SELF && transitive_chain < 100 {
+                current_verifier_id = match verified_by_map.get(&current_verifier_id) {
+                    Some(id) => *id,
+                    None => {
+                        // The chain ends here, probably because some verification was done
+                        // before we started recording verifiers.
+                        // It's unclear how long the chain really is.
+                        transitive_chain = 0;
+                        break;
+                    }
+                };
+                if bot_ids.contains(&current_verifier_id) {
+                    has_bot = true;
+                }
+                transitive_chain = transitive_chain.saturating_add(1);
+            }
+
+            if transitive_chain > 0 {
+                contact.transitive_chain = Some(transitive_chain);
+            }
+
+            if has_bot {
+                contact.verified = VerifiedStatus::TransitiveViaBot;
+            }
+        }
+    }
+
+    // Fill direct_chat
+    for contact in &mut contacts {
+        let direct_chat = context
+            .sql
+            .exists(
+                "SELECT COUNT(*)
+                FROM chats_contacts cc INNER JOIN chats
+                WHERE cc.contact_id=? AND chats.type=?",
+                (contact.id, Chattype::Single),
+            )
+            .await?;
+        contact.direct_chat = direct_chat;
+    }
+
+    Ok(contacts)
+}
+
+/// - `last_msg_id`: The last msg_id that was already counted in the previous stats.
+///   Only messages newer than that will be counted.
+/// - `one_one_chats`: If true, only messages in 1:1 chats are counted.
+///   If false, only messages in other chats (groups and broadcast channels) are counted.
+async fn get_message_stats(context: &Context) -> Result<BTreeMap<Chattype, MessageStats>> {
+    let mut map: BTreeMap<Chattype, MessageStats> = context
+        .sql
+        .query_map(
+            "SELECT chattype, verified, unverified_encrypted, unencrypted, only_to_self
+            FROM stats_msgs",
+            (),
+            |row| {
+                let chattype: Chattype = row.get(0)?;
+                let verified: u32 = row.get(1)?;
+                let unverified_encrypted: u32 = row.get(2)?;
+                let unencrypted: u32 = row.get(3)?;
+                let only_to_self: u32 = row.get(4)?;
+                let message_stats = MessageStats {
+                    verified,
+                    unverified_encrypted,
+                    unencrypted,
+                    only_to_self,
+                };
+                Ok((chattype, message_stats))
+            },
+            |rows| Ok(rows.collect::<rusqlite::Result<BTreeMap<_, _>>>()?),
+        )
+        .await?;
+
+    // Fill zeroes if a chattype wasn't present:
+    for chattype in [Chattype::Group, Chattype::Single, Chattype::OutBroadcast] {
+        map.entry(chattype).or_default();
+    }
+
+    Ok(map)
+}
+
+pub(crate) async fn update_message_stats(context: &Context) -> Result<()> {
+    for chattype in [Chattype::Single, Chattype::Group, Chattype::OutBroadcast] {
+        update_message_stats_inner(context, chattype).await?;
+    }
+    context
+        .set_config_internal(Config::StatsLastUpdate, Some(&time().to_string()))
+        .await?;
+    Ok(())
+}
+
+async fn update_message_stats_inner(context: &Context, chattype: Chattype) -> Result<()> {
+    let stats_bot_chat_id = get_stats_chat_id(context).await?;
+
+    let trans_fn = |t: &mut rusqlite::Transaction| {
+        // The ID of the last msg that was already counted in the previously sent stats.
+        // Only newer messages will be counted in the current statistics.
+        let last_counted_msg_id: u32 = t
+            .query_row(
+                "SELECT last_counted_msg_id FROM stats_msgs WHERE chattype=?",
+                (chattype,),
+                |row| row.get(0),
+            )
+            .optional()?
+            .unwrap_or(0);
+        t.execute(
+            "UPDATE stats_msgs
+            SET last_counted_msg_id=(SELECT MAX(id) FROM msgs)
+            WHERE chattype=?",
+            (chattype,),
+        )?;
+
+        // This table will hold all empty chats,
+        // i.e. all chats that do not contain any members except for self.
+        // Messages in these chats are not actually sent out.
+        t.execute(
+            "CREATE TEMP TABLE temp.empty_chats (
+                id INTEGER PRIMARY KEY
+            ) STRICT",
+            (),
+        )?;
+
+        // id>9 because chat ids 0..9 are "special" chats like the trash chat,
+        // and contact ids 0..9 are "special" contact ids like the 'device'.
+        t.execute(
+            "INSERT INTO temp.empty_chats
+            SELECT id FROM chats
+            WHERE id>9 AND NOT EXISTS(
+                SELECT *
+                FROM contacts, chats_contacts
+                WHERE chats_contacts.contact_id=contacts.id AND chats_contacts.chat_id=chats.id
+				AND contacts.id>9
+            )",
+            (),
+        )?;
+
+        // This table will hold all verified chats,
+        // i.e. all chats that only contain verified contacts.
+        t.execute(
+            "CREATE TEMP TABLE temp.verified_chats (
+                id INTEGER PRIMARY KEY
+            ) STRICT",
+            (),
+        )?;
+
+        // Verified chats are chats that are not empty,
+        // and do not contain any unverified contacts
+        t.execute(
+            "INSERT INTO temp.verified_chats
+            SELECT id FROM chats
+            WHERE id>9
+            AND id NOT IN (SELECT id FROM temp.empty_chats)
+            AND NOT EXISTS(
+                SELECT *
+                FROM contacts, chats_contacts
+                WHERE chats_contacts.contact_id=contacts.id AND chats_contacts.chat_id=chats.id
+				AND contacts.id>9
+				AND contacts.verifier=0
+            )",
+            (),
+        )?;
+
+        // This table will hold all 1:1 chats.
+        t.execute(
+            "CREATE TEMP TABLE temp.chat_with_correct_type (
+                id INTEGER PRIMARY KEY
+            ) STRICT",
+            (),
+        )?;
+
+        t.execute(
+            "INSERT INTO temp.chat_with_correct_type
+            SELECT id FROM chats
+            WHERE type=?;",
+            (chattype,),
+        )?;
+
+        // - `from_id=?` is to count only outgoing messages.
+        // - `chat_id<>?` excludes the chat with the statistics bot itself,
+        // - `id>?` excludes messages that were already counted in the previously sent statistics, or messages sent before the config was enabled
+        // - `hidden=0` excludes hidden system messages, which are not actually shown to the user.
+        //   Note that reactions are also not counted as a message.
+        // - `chat_id>9` excludes messages in the 'Trash' chat, which is an internal chat assigned to messages that are not shown to the user
+        let general_requirements = "id>? AND from_id=? AND chat_id<>?
+            AND hidden=0 AND chat_id>9 AND chat_id IN temp.chat_with_correct_type"
+            .to_string();
+        let params = (last_counted_msg_id, ContactId::SELF, stats_bot_chat_id);
+
+        let verified: u32 = t.query_row(
+            &format!(
+                "SELECT COUNT(*) FROM msgs
+                WHERE chat_id IN temp.verified_chats
+                AND {general_requirements}"
+            ),
+            params,
+            |row| row.get(0),
+        )?;
+
+        let unverified_encrypted: u32 = t.query_row(
+            &format!(
+                // (param GLOB '*\nc=1*' OR param GLOB 'c=1*') matches all messages that are end-to-end encrypted
+                "SELECT COUNT(*) FROM msgs
+                WHERE chat_id NOT IN temp.verified_chats AND chat_id NOT IN temp.empty_chats
+                AND (param GLOB '*\nc=1*' OR param GLOB 'c=1*')
+                AND {general_requirements}"
+            ),
+            params,
+            |row| row.get(0),
+        )?;
+
+        let unencrypted: u32 = t.query_row(
+            &format!(
+                "SELECT COUNT(*) FROM msgs
+                WHERE chat_id NOT IN temp.verified_chats AND chat_id NOT IN temp.empty_chats
+                AND NOT (param GLOB '*\nc=1*' OR param GLOB 'c=1*')
+                AND {general_requirements}"
+            ),
+            params,
+            |row| row.get(0),
+        )?;
+
+        let only_to_self: u32 = t.query_row(
+            &format!(
+                "SELECT COUNT(*) FROM msgs
+                WHERE chat_id IN temp.empty_chats
+                AND {general_requirements}"
+            ),
+            params,
+            |row| row.get(0),
+        )?;
+
+        t.execute("DROP TABLE temp.verified_chats", ())?;
+        t.execute("DROP TABLE temp.empty_chats", ())?;
+        t.execute("DROP TABLE temp.chat_with_correct_type", ())?;
+
+        t.execute(
+            "INSERT INTO stats_msgs(chattype) VALUES (?)
+            ON CONFLICT(chattype) DO NOTHING",
+            (chattype,),
+        )?;
+        t.execute(
+            "UPDATE stats_msgs SET
+            verified=verified+?,
+            unverified_encrypted=unverified_encrypted+?,
+            unencrypted=unencrypted+?,
+            only_to_self=only_to_self+?
+            WHERE chattype=?",
+            (
+                verified,
+                unverified_encrypted,
+                unencrypted,
+                only_to_self,
+                chattype,
+            ),
+        )?;
+
+        Ok(())
+    };
+
+    context.sql.transaction(trans_fn).await?;
+
+    Ok(())
+}
+
+pub(crate) async fn count_securejoin_ux_info(
+    context: &Context,
+    source: Option<SecurejoinSource>,
+    uipath: Option<SecurejoinUiPath>,
+) -> Result<()> {
+    if !should_send_stats(context).await? {
+        return Ok(());
+    }
+
+    let source = source
+        .context("Missing securejoin source")
+        .log_err(context)
+        .unwrap_or(SecurejoinSource::Unknown);
+
+    // We only get a UI path if the source is a QR code scan,
+    // a loaded image, or a link pasted from the QR code,
+    // so, no need to log an error if `uipath` is None:
+    let uipath = uipath.unwrap_or(SecurejoinUiPath::Unknown);
+
+    context
+        .sql
+        .transaction(|conn| {
+            conn.execute(
+                "INSERT INTO stats_securejoin_sources VALUES (?, 1)
+                ON CONFLICT (source) DO UPDATE SET count=count+1;",
+                (source.to_u32(),),
+            )?;
+
+            conn.execute(
+                "INSERT INTO stats_securejoin_uipaths VALUES (?, 1)
+                ON CONFLICT (uipath) DO UPDATE SET count=count+1;",
+                (uipath.to_u32(),),
+            )?;
+            Ok(())
+        })
+        .await?;
+
+    Ok(())
+}
+
+async fn get_securejoin_source_stats(context: &Context) -> Result<SecurejoinSources> {
+    let map = context
+        .sql
+        .query_map(
+            "SELECT source, count FROM stats_securejoin_sources",
+            (),
+            |row| {
+                let source: SecurejoinSource = row.get(0)?;
+                let count: u32 = row.get(1)?;
+                Ok((source, count))
+            },
+            |rows| Ok(rows.collect::<rusqlite::Result<BTreeMap<_, _>>>()?),
+        )
+        .await?;
+
+    let stats = SecurejoinSources {
+        unknown: *map.get(&SecurejoinSource::Unknown).unwrap_or(&0),
+        external_link: *map.get(&SecurejoinSource::ExternalLink).unwrap_or(&0),
+        internal_link: *map.get(&SecurejoinSource::InternalLink).unwrap_or(&0),
+        clipboard: *map.get(&SecurejoinSource::Clipboard).unwrap_or(&0),
+        image_loaded: *map.get(&SecurejoinSource::ImageLoaded).unwrap_or(&0),
+        scan: *map.get(&SecurejoinSource::Scan).unwrap_or(&0),
+    };
+
+    Ok(stats)
+}
+
+async fn get_securejoin_uipath_stats(context: &Context) -> Result<SecurejoinUiPaths> {
+    let map = context
+        .sql
+        .query_map(
+            "SELECT uipath, count FROM stats_securejoin_uipaths",
+            (),
+            |row| {
+                let uipath: SecurejoinUiPath = row.get(0)?;
+                let count: u32 = row.get(1)?;
+                Ok((uipath, count))
+            },
+            |rows| Ok(rows.collect::<rusqlite::Result<BTreeMap<_, _>>>()?),
+        )
+        .await?;
+
+    let stats = SecurejoinUiPaths {
+        other: *map.get(&SecurejoinUiPath::Unknown).unwrap_or(&0),
+        qr_icon: *map.get(&SecurejoinUiPath::QrIcon).unwrap_or(&0),
+        new_contact: *map.get(&SecurejoinUiPath::NewContact).unwrap_or(&0),
+    };
+
+    Ok(stats)
+}
+
+pub(crate) async fn count_securejoin_invite(context: &Context, invite: &QrInvite) -> Result<()> {
+    if !should_send_stats(context).await? {
+        return Ok(());
+    }
+
+    let contact = Contact::get_by_id(context, invite.contact_id()).await?;
+
+    // If the contact was created just now by the QR code scan,
+    // (or if a contact existed in the database
+    // but it was not visible in the contacts list in the UI
+    // e.g. because it's a past contact of a group we're in),
+    // then its origin is UnhandledSecurejoinQrScan.
+    let already_existed = contact.origin > Origin::UnhandledSecurejoinQrScan;
+
+    // Check whether the contact was verified already before the QR scan.
+    let already_verified = contact.is_verified(context).await?;
+
+    let typ = match invite {
+        QrInvite::Contact { .. } => "contact",
+        QrInvite::Group { .. } => "group",
+    };
+
+    context
+        .sql
+        .execute(
+            "INSERT INTO stats_securejoin_invites (already_existed, already_verified, type)
+            VALUES (?, ?, ?)",
+            (already_existed, already_verified, typ),
+        )
+        .await?;
+
+    Ok(())
+}
+
+async fn get_securejoin_invite_stats(context: &Context) -> Result<Vec<JoinedInvite>> {
+    let qr_scans: Vec<JoinedInvite> = context
+        .sql
+        .query_map(
+            "SELECT already_existed, already_verified, type FROM stats_securejoin_invites",
+            (),
+            |row| {
+                let already_existed: bool = row.get(0)?;
+                let already_verified: bool = row.get(1)?;
+                let typ: String = row.get(2)?;
+
+                Ok(JoinedInvite {
+                    already_existed,
+                    already_verified,
+                    typ,
+                })
+            },
+            |rows| {
+                rows.collect::<std::result::Result<Vec<_>, _>>()
+                    .map_err(Into::into)
+            },
+        )
+        .await?;
+
+    Ok(qr_scans)
+}
+
+#[cfg(test)]
+mod stats_tests;
diff --git a/src/stats/stats_tests.rs b/src/stats/stats_tests.rs
new file mode 100644
index 000000000..ab1ab9c29
--- /dev/null
+++ b/src/stats/stats_tests.rs
@@ -0,0 +1,595 @@
+use std::time::Duration;
+
+use super::*;
+use crate::chat::{
+    Chat, create_broadcast, create_group, create_group_unencrypted, get_chat_contacts,
+};
+use crate::mimeparser::SystemMessage;
+use crate::qr::check_qr;
+use crate::securejoin::{get_securejoin_qr, join_securejoin, join_securejoin_with_ux_info};
+use crate::test_utils::{TestContext, TestContextManager, get_chat_msg};
+use crate::tools::SystemTime;
+use pretty_assertions::assert_eq;
+use serde_json::{Number, Value};
+
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_maybe_send_stats() -> Result<()> {
+    let alice = &TestContext::new_alice().await;
+
+    // Can't use `set_config()` here, because this would directly send the statistics,
+    // and we wouldn't know the chat id
+    alice
+        .set_config_internal(Config::StatsSending, Some("1"))
+        .await?;
+
+    let chat_id = maybe_send_stats(alice).await?.unwrap();
+    let msg = get_chat_msg(alice, chat_id, 0, 2).await;
+    assert_eq!(msg.get_info_type(), SystemMessage::ChatE2ee);
+
+    let chat = Chat::load_from_db(alice, chat_id).await?;
+    assert!(chat.is_encrypted(alice).await?);
+    let contacts = get_chat_contacts(alice, chat_id).await?;
+    assert_eq!(contacts.len(), 1);
+    let contact = Contact::get_by_id(alice, contacts[0]).await?;
+    assert!(contact.is_verified(alice).await?);
+
+    let msg = get_chat_msg(alice, chat_id, 1, 2).await;
+    assert_eq!(msg.get_filename().unwrap(), "statistics.txt");
+
+    let stats = tokio::fs::read(msg.get_file(alice).unwrap()).await?;
+    let stats = std::str::from_utf8(&stats)?;
+    println!("\nEmpty account:\n{stats}\n");
+    assert!(stats.contains(r#""contact_stats": []"#));
+
+    let r: serde_json::Value = serde_json::from_str(stats)?;
+    assert_eq!(
+        r.get("contact_stats").unwrap(),
+        &serde_json::Value::Array(vec![])
+    );
+    assert_eq!(r.get("core_version").unwrap(), get_version_str());
+
+    assert_eq!(maybe_send_stats(alice).await?, None);
+
+    Ok(())
+}
+
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_rewound_time() -> Result<()> {
+    let alice = &TestContext::new_alice().await;
+    alice.set_config_bool(Config::StatsSending, true).await?;
+
+    // Enabling StatsSending directly sends the first statistics,
+    // so that the user immediately sees the result of enabling it:
+    assert!(maybe_send_stats(alice).await?.is_none());
+    let sent = alice.pop_sent_msg().await;
+    assert_eq!(
+        sent.load_from_db().await.get_filename().unwrap(),
+        "statistics.txt"
+    );
+
+    const EIGHT_DAYS: Duration = Duration::from_secs(3600 * 24 * 14);
+    SystemTime::shift(EIGHT_DAYS);
+
+    maybe_send_stats(alice).await?.unwrap();
+
+    // The system's time is rewound
+    SystemTime::shift_back(EIGHT_DAYS);
+
+    assert!(maybe_send_stats(alice).await?.is_none());
+
+    // After eight days pass again, stats are sent again
+    SystemTime::shift(EIGHT_DAYS);
+    maybe_send_stats(alice).await?.unwrap();
+
+    Ok(())
+}
+
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_stats_one_contact() -> Result<()> {
+    let mut tcm = TestContextManager::new();
+    let alice = &tcm.alice().await;
+    let bob = &tcm.bob().await;
+    alice.set_config_bool(Config::StatsSending, true).await?;
+
+    let stats = get_stats(alice).await?;
+    let r: serde_json::Value = serde_json::from_str(&stats)?;
+
+    tcm.send_recv_accept(bob, alice, "Hi!").await;
+
+    let stats = get_stats(alice).await?;
+    println!("\nWith Bob:\n{stats}\n");
+    let r2: serde_json::Value = serde_json::from_str(&stats)?;
+
+    assert_eq!(
+        r.get("key_create_timestamps").unwrap(),
+        r2.get("key_create_timestamps").unwrap()
+    );
+    assert_eq!(r.get("stats_id").unwrap(), r2.get("stats_id").unwrap());
+    let contact_stats = r2.get("contact_stats").unwrap().as_array().unwrap();
+    assert_eq!(contact_stats.len(), 1);
+    let contact_info = &contact_stats[0];
+    assert!(contact_info.get("bot").is_none());
+    assert_eq!(
+        contact_info.get("direct_chat").unwrap(),
+        &serde_json::Value::Bool(true)
+    );
+    assert!(contact_info.get("transitive_chain").is_none(),);
+    assert_eq!(
+        contact_info.get("verified").unwrap(),
+        &serde_json::Value::String("Opportunistic".to_string())
+    );
+    assert_eq!(
+        contact_info.get("new").unwrap(),
+        &serde_json::Value::Bool(true)
+    );
+
+    Ok(())
+}
+
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_message_stats() -> Result<()> {
+    #[track_caller]
+    fn check_stats(stats: &str, expected: &BTreeMap<Chattype, MessageStats>) {
+        let actual: serde_json::Value = serde_json::from_str(stats).unwrap();
+        let actual = &actual["message_stats"];
+
+        let expected = serde_json::to_string_pretty(&expected).unwrap();
+        let expected: serde_json::Value = serde_json::from_str(&expected).unwrap();
+
+        assert_eq!(actual, &expected);
+    }
+
+    async fn update_get_stats(context: &Context) -> String {
+        update_message_stats(context).await.unwrap();
+        get_stats(context).await.unwrap()
+    }
+
+    let mut tcm = TestContextManager::new();
+    let alice = &tcm.alice().await;
+    let bob = &tcm.bob().await;
+    // Can't use `set_config()` here, because this would directly send the statistics
+    alice
+        .set_config_internal(Config::StatsSending, Some("1"))
+        .await?;
+    let email_chat = alice.create_email_chat(bob).await;
+    let encrypted_chat = alice.create_chat(bob).await;
+
+    let mut expected: BTreeMap<Chattype, MessageStats> = BTreeMap::from_iter([
+        (Chattype::Single, MessageStats::default()),
+        (Chattype::Group, MessageStats::default()),
+        (Chattype::OutBroadcast, MessageStats::default()),
+    ]);
+
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    alice.send_text(email_chat.id, "foo").await;
+    expected.get_mut(&Chattype::Single).unwrap().unencrypted += 1;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    alice.send_text(encrypted_chat.id, "foo").await;
+    expected
+        .get_mut(&Chattype::Single)
+        .unwrap()
+        .unverified_encrypted += 1;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    alice.send_text(encrypted_chat.id, "foo").await;
+    expected
+        .get_mut(&Chattype::Single)
+        .unwrap()
+        .unverified_encrypted += 1;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    let group = alice.create_group_with_members("Pizza", &[bob]).await;
+    alice.send_text(group, "foo").await;
+    expected
+        .get_mut(&Chattype::Group)
+        .unwrap()
+        .unverified_encrypted += 1;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    tcm.execute_securejoin(alice, bob).await;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    alice.send_text(alice.get_self_chat().await.id, "foo").await;
+    expected.get_mut(&Chattype::Single).unwrap().only_to_self += 1;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    let empty_group = create_group(alice, "Notes").await?;
+    alice.send_text(empty_group, "foo").await;
+    expected.get_mut(&Chattype::Group).unwrap().only_to_self += 1;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    let empty_unencrypted = create_group_unencrypted(alice, "Email thread").await?;
+    alice.send_text(empty_unencrypted, "foo").await;
+    expected.get_mut(&Chattype::Group).unwrap().only_to_self += 1;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    let group = alice.create_group_with_members("Pizza 2", &[bob]).await;
+    alice.send_text(group, "foo").await;
+    expected.get_mut(&Chattype::Group).unwrap().verified += 1;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    let empty_broadcast = create_broadcast(alice, "Channel".to_string()).await?;
+    alice.send_text(empty_broadcast, "foo").await;
+    expected
+        .get_mut(&Chattype::OutBroadcast)
+        .unwrap()
+        .only_to_self += 1;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    // Incoming messages are not counted:
+    let rcvd = tcm.send_recv(bob, alice, "bar").await;
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    // Reactions are not counted:
+    crate::reaction::send_reaction(alice, rcvd.id, "👍")
+        .await
+        .unwrap();
+    check_stats(&update_get_stats(alice).await, &expected);
+
+    let before_sending = get_stats(alice).await.unwrap();
+
+    let stats = send_and_read_stats(alice).await;
+    // The stats are supposed not to have changed yet
+    assert_eq!(before_sending, stats);
+
+    // Shift by 8 days so that the next stats-sending is due:
+    SystemTime::shift(Duration::from_secs(8 * 24 * 3600));
+
+    let stats = send_and_read_stats(alice).await;
+    assert_eq!(before_sending, stats);
+
+    check_stats(&stats, &expected);
+
+    SystemTime::shift(Duration::from_secs(8 * 24 * 3600));
+    tcm.send_recv(alice, bob, "Hi").await;
+    expected.get_mut(&Chattype::Single).unwrap().verified += 1;
+    update_message_stats(alice).await?;
+    update_message_stats(alice).await?;
+    tcm.send_recv(alice, bob, "Hi").await;
+    expected.get_mut(&Chattype::Single).unwrap().verified += 1;
+    tcm.send_recv(alice, bob, "Hi").await;
+    expected.get_mut(&Chattype::Single).unwrap().verified += 1;
+
+    check_stats(&send_and_read_stats(alice).await, &expected);
+
+    Ok(())
+}
+
+async fn send_and_read_stats(context: &TestContext) -> String {
+    let chat_id = maybe_send_stats(context).await.unwrap().unwrap();
+    let msg = context.get_last_msg_in(chat_id).await;
+    assert_eq!(msg.get_filename().unwrap(), "statistics.txt");
+
+    let stats = tokio::fs::read(msg.get_file(context).unwrap())
+        .await
+        .unwrap();
+    String::from_utf8(stats).unwrap()
+}
+
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_stats_securejoin_sources() -> Result<()> {
+    async fn check_stats(context: &TestContext, expected: &SecurejoinSources) {
+        let stats = get_stats(context).await.unwrap();
+        let actual: serde_json::Value = serde_json::from_str(&stats).unwrap();
+        let actual = &actual["securejoin_sources"];
+
+        let expected = serde_json::to_string_pretty(&expected).unwrap();
+        let expected: serde_json::Value = serde_json::from_str(&expected).unwrap();
+
+        assert_eq!(actual, &expected);
+    }
+
+    let mut tcm = TestContextManager::new();
+    let alice = &tcm.alice().await;
+    let bob = &tcm.bob().await;
+    alice.set_config_bool(Config::StatsSending, true).await?;
+
+    let mut expected = SecurejoinSources {
+        unknown: 0,
+        external_link: 0,
+        internal_link: 0,
+        clipboard: 0,
+        image_loaded: 0,
+        scan: 0,
+    };
+
+    check_stats(alice, &expected).await;
+
+    let qr = get_securejoin_qr(bob, None).await?;
+
+    join_securejoin(alice, &qr).await?;
+    expected.unknown += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin(alice, &qr).await?;
+    expected.unknown += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, Some(SecurejoinSource::Clipboard), None).await?;
+    expected.clipboard += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, Some(SecurejoinSource::ExternalLink), None).await?;
+    expected.external_link += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, Some(SecurejoinSource::InternalLink), None).await?;
+    expected.internal_link += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, Some(SecurejoinSource::ImageLoaded), None).await?;
+    expected.image_loaded += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, Some(SecurejoinSource::Scan), None).await?;
+    expected.scan += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, Some(SecurejoinSource::Clipboard), None).await?;
+    expected.clipboard += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, Some(SecurejoinSource::Clipboard), None).await?;
+    expected.clipboard += 1;
+    check_stats(alice, &expected).await;
+
+    Ok(())
+}
+
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_stats_securejoin_uipaths() -> Result<()> {
+    async fn check_stats(context: &TestContext, expected: &SecurejoinUiPaths) {
+        let stats = get_stats(context).await.unwrap();
+        let actual: serde_json::Value = serde_json::from_str(&stats).unwrap();
+        let actual = &actual["securejoin_uipaths"];
+
+        let expected = serde_json::to_string_pretty(&expected).unwrap();
+        let expected: serde_json::Value = serde_json::from_str(&expected).unwrap();
+
+        assert_eq!(actual, &expected);
+    }
+
+    let mut tcm = TestContextManager::new();
+    let alice = &tcm.alice().await;
+    let bob = &tcm.bob().await;
+    alice.set_config_bool(Config::StatsSending, true).await?;
+
+    let mut expected = SecurejoinUiPaths {
+        other: 0,
+        qr_icon: 0,
+        new_contact: 0,
+    };
+
+    check_stats(alice, &expected).await;
+
+    let qr = get_securejoin_qr(bob, None).await?;
+
+    join_securejoin(alice, &qr).await?;
+    expected.other += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin(alice, &qr).await?;
+    expected.other += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, None, Some(SecurejoinUiPath::NewContact)).await?;
+    expected.new_contact += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, None, Some(SecurejoinUiPath::NewContact)).await?;
+    expected.new_contact += 1;
+    check_stats(alice, &expected).await;
+
+    join_securejoin_with_ux_info(alice, &qr, None, Some(SecurejoinUiPath::QrIcon)).await?;
+    expected.qr_icon += 1;
+    check_stats(alice, &expected).await;
+
+    Ok(())
+}
+
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_stats_securejoin_invites() -> Result<()> {
+    async fn check_stats(context: &TestContext, expected: &[JoinedInvite]) {
+        let stats = get_stats(context).await.unwrap();
+        let actual: serde_json::Value = serde_json::from_str(&stats).unwrap();
+        let actual = &actual["securejoin_invites"];
+
+        let expected = serde_json::to_string_pretty(&expected).unwrap();
+        let expected: serde_json::Value = serde_json::from_str(&expected).unwrap();
+
+        assert_eq!(actual, &expected);
+    }
+
+    let mut tcm = TestContextManager::new();
+    let alice = &tcm.alice().await;
+    let bob = &tcm.bob().await;
+    let charlie = &tcm.charlie().await;
+    alice.set_config_bool(Config::StatsSending, true).await?;
+
+    let mut expected = vec![];
+
+    check_stats(alice, &expected).await;
+
+    let qr = get_securejoin_qr(bob, None).await?;
+
+    // The UI will call `check_qr()` first, which must not make the stats wrong:
+    check_qr(alice, &qr).await?;
+    tcm.exec_securejoin_qr(alice, bob, &qr).await;
+    expected.push(JoinedInvite {
+        already_existed: false,
+        already_verified: false,
+        typ: "contact".to_string(),
+    });
+    check_stats(alice, &expected).await;
+
+    check_qr(alice, &qr).await?;
+    tcm.exec_securejoin_qr(alice, bob, &qr).await;
+    expected.push(JoinedInvite {
+        already_existed: true,
+        already_verified: true,
+        typ: "contact".to_string(),
+    });
+    check_stats(alice, &expected).await;
+
+    let group_id = create_group(bob, "Group chat").await?;
+    let qr = get_securejoin_qr(bob, Some(group_id)).await?;
+
+    check_qr(alice, &qr).await?;
+    tcm.exec_securejoin_qr(alice, bob, &qr).await;
+    expected.push(JoinedInvite {
+        already_existed: true,
+        already_verified: true,
+        typ: "group".to_string(),
+    });
+    check_stats(alice, &expected).await;
+
+    // A contact with Charlie exists already:
+    alice.add_or_lookup_contact(charlie).await;
+    let group_id = create_group(charlie, "Group chat 2").await?;
+    let qr = get_securejoin_qr(charlie, Some(group_id)).await?;
+
+    check_qr(alice, &qr).await?;
+    tcm.exec_securejoin_qr(alice, bob, &qr).await;
+    expected.push(JoinedInvite {
+        already_existed: true,
+        already_verified: false,
+        typ: "group".to_string(),
+    });
+    check_stats(alice, &expected).await;
+
+    Ok(())
+}
+
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_stats_is_chatmail() -> Result<()> {
+    let alice = &TestContext::new_alice().await;
+    alice.set_config_bool(Config::StatsSending, true).await?;
+
+    let r = get_stats(alice).await?;
+    let r: serde_json::Value = serde_json::from_str(&r)?;
+    assert_eq!(r.get("is_chatmail").unwrap().as_bool().unwrap(), false);
+
+    alice.set_config_bool(Config::IsChatmail, true).await?;
+
+    let r = get_stats(alice).await?;
+    let r: serde_json::Value = serde_json::from_str(&r)?;
+    assert_eq!(r.get("is_chatmail").unwrap().as_bool().unwrap(), true);
+
+    Ok(())
+}
+
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_stats_key_creation_timestamp() -> Result<()> {
+    // Alice uses a pregenerated key. It was created at this timestamp:
+    const ALICE_KEY_CREATION_TIME: u128 = 1582855645;
+
+    let alice = &TestContext::new_alice().await;
+    alice.set_config_bool(Config::StatsSending, true).await?;
+
+    let r = get_stats(alice).await?;
+    let r: serde_json::Value = serde_json::from_str(&r)?;
+    let key_create_timestamps = r.get("key_create_timestamps").unwrap().as_array().unwrap();
+    assert_eq!(
+        key_create_timestamps,
+        &vec![Value::Number(
+            Number::from_u128(ALICE_KEY_CREATION_TIME).unwrap()
+        )]
+    );
+
+    Ok(())
+}
+
+/// We record the timestamp when StatsSending is enabled.
+/// If it's disabled and then enabled again, we also record these timestamps.
+/// This test enables, disables, and reenables StatsSending,
+/// and checks that the timestamps are recorded correctly.
+#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+async fn test_stats_enable_disable_timestamps() -> Result<()> {
+    async fn get_timestamps(context: &TestContext) -> (Vec<i64>, Vec<i64>) {
+        let stats = get_stats(context).await.unwrap();
+        let stats: serde_json::Value = serde_json::from_str(&stats).unwrap();
+        let enabled_ts = &stats["sending_enabled_timestamps"];
+        let disabled_ts = &stats["sending_disabled_timestamps"];
+
+        let enabled_ts = enabled_ts
+            .as_array()
+            .unwrap()
+            .iter()
+            .map(|v| v.as_i64().unwrap())
+            .collect();
+        let disabled_ts = disabled_ts
+            .as_array()
+            .unwrap()
+            .iter()
+            .map(|v| v.as_i64().unwrap())
+            .collect();
+
+        (enabled_ts, disabled_ts)
+    }
+
+    let alice = &TestContext::new_alice().await;
+
+    // ============================== Enable the setting, and check corresponding timestamp ==============================
+    let enabled_min = time();
+    alice.set_config_bool(Config::StatsSending, true).await?;
+    let enabled_max = time();
+
+    let (enabled_ts, disabled_ts) = get_timestamps(alice).await;
+
+    // The enabling timestamp was inbetween `enabled_min` and `enabled_max`:
+    assert_eq!(enabled_ts.len(), 1);
+    assert!(enabled_ts[0] >= enabled_min);
+    assert!(enabled_ts[0] <= enabled_max);
+
+    assert!(disabled_ts.is_empty());
+
+    // Enabling again should not make a difference
+    alice.set_config_bool(Config::StatsSending, true).await?;
+    SystemTime::shift(Duration::from_secs(10));
+    alice.set_config_bool(Config::StatsSending, true).await?;
+    assert_eq!(
+        get_timestamps(alice).await,
+        (enabled_ts.clone(), disabled_ts.clone())
+    );
+
+    // ============================== Disable the setting, and check corresponding timestamp ==============================
+    let disabled_min = time();
+    alice.set_config_bool(Config::StatsSending, false).await?;
+    let disabled_max = time();
+
+    let (new_enabled_ts, new_disabled_ts) = get_timestamps(alice).await;
+
+    assert_eq!(new_enabled_ts, enabled_ts); // The timestamp of enabling didn't change
+
+    // The disabling timestamp was inbetween `disabled_min` and `disabled_max`:
+    assert_eq!(new_disabled_ts.len(), 1);
+    assert!(new_disabled_ts[0] >= disabled_min);
+    assert!(new_disabled_ts[0] <= disabled_max);
+
+    // The time should have advanced in the meantime (because of SystemTime::shift()):
+    assert_ne!(new_disabled_ts[0], enabled_ts[0]);
+
+    // ============================== Enable the setting again ==============================
+    SystemTime::shift(Duration::from_secs(10));
+    let enabled_min = time();
+    alice.set_config_bool(Config::StatsSending, true).await?;
+    let enabled_max = time();
+
+    let (newer_enabled_ts, newer_disabled_ts) = get_timestamps(alice).await;
+
+    // The timestamp of disabling didn't change:
+    assert_eq!(newer_disabled_ts, new_disabled_ts);
+
+    // The enabling timestamp was inbetween `enabled_min` and `enabled_max`:
+    assert_eq!(newer_enabled_ts.len(), 2);
+    assert!(newer_enabled_ts[1] >= enabled_min);
+    assert!(newer_enabled_ts[1] <= enabled_max);
+    assert_eq!(newer_enabled_ts[0], new_enabled_ts[0]);
+
+    // The time should have advanced in the meantime (because of SystemTime::shift()):
+    assert_ne!(newer_disabled_ts[0], newer_enabled_ts[1]);
+
+    Ok(())
+}
diff --git a/src/stock_str.rs b/src/stock_str.rs
index 381e9a81f..89266471a 100644
--- a/src/stock_str.rs
+++ b/src/stock_str.rs
@@ -431,6 +431,11 @@ pub enum StockMessage {
 
     #[strum(props(fallback = "Scan to join channel %1$s"))]
     SecureJoinBrodcastQRDescription = 201,
+
+    #[strum(props(
+        fallback = "The attachment contains anonymous usage statistics, which helps us improve Delta Chat. Thank you!"
+    ))]
+    StatsMsgBody = 210,
 }
 
 impl StockMessage {
@@ -1262,6 +1267,11 @@ pub(crate) async fn unencrypted_email(context: &Context, provider: &str) -> Stri
         .replace1(provider)
 }
 
+/// Stock string: `The attachment contains anonymous usage statistics, which helps us improve Delta Chat. Thank you!`
+pub(crate) async fn stats_msg_body(context: &Context) -> String {
+    translated(context, StockMessage::StatsMsgBody).await
+}
+
 pub(crate) async fn aeap_explanation_and_link(
     context: &Context,
     old_addr: &str,
```

I later wrote a fix that made the generated IDs URL-safe for use in Qualtrics:

```diff
diff --git a/src/stats.rs b/src/stats.rs
index 33d9f7452..9f97e5a48 100644
--- a/src/stats.rs
+++ b/src/stats.rs
@@ -9,6 +9,7 @@
 use deltachat_derive::FromSql;
 use num_traits::ToPrimitive;
 use pgp::types::PublicKeyTrait;
+use rand::distr::SampleString as _;
 use rusqlite::OptionalExtension;
 use serde::Serialize;
 
@@ -21,7 +22,7 @@
 use crate::log::LogExt;
 use crate::message::{Message, Viewtype};
 use crate::securejoin::QrInvite;
-use crate::tools::{create_id, time};
+use crate::tools::time;
 
 pub(crate) const STATISTICS_BOT_EMAIL: &str = "self_reporting@testrun.org";
 const STATISTICS_BOT_VCARD: &str = include_str!("../assets/statistics-bot.vcf");
@@ -390,7 +391,9 @@ pub(crate) async fn stats_id(context: &Context) -> Result<String> {
     Ok(match context.get_config(Config::StatsId).await? {
         Some(id) => id,
         None => {
-            let id = create_id();
+            let id = rand::distr::Alphabetic
+                .sample_string(&mut rand::rng(), 25)
+                .to_lowercase();
             context
                 .set_config_internal(Config::StatsId, Some(&id))
                 .await?;
```

## Bot that collects the statistics

The complete, runnable bot is at https://github.com/deltachat/self_reporting_bot. GitHub user `adbenitez` ported it for me from an old bot API to the new API, wrote the `pyproject.toml` file, and improved the logging.

This is the final code for the bot itself (`self_reporting_bot.py`):

```python
import logging
import os.path
from pathlib import Path
import time
import json

from deltabot_cli import BotCli
from deltachat2 import EventType, events
from deltachat2.types import SpecialContactId
from rich.logging import RichHandler

REPORTS_DIR = Path("reports")

cli = BotCli("self_reporting_bot")
logging.basicConfig(
    level=logging.INFO,
    format="%(message)s",
    handlers=[RichHandler(show_time=False, show_path=False)],
)


@cli.on(events.NewMessage)
def on_new_message(bot, accid, event):
    chatid = event.msg.chat_id
    msg = event.msg
    try:
        if msg.text.startswith("core_version "):
            bot.rpc.misc_send_text_message(
                accid,
                chatid,
                "You are using an outdated version of Delta Chat. Please update and try again.",
            )
            return

        if msg.file_name != "statistics.txt":
            raise ValueError("Message doesn't contain statistics.txt")

        with open(msg.file) as file:
            new_data = json.load(file)

        stats_id = new_data["stats_id"]
        if any(c not in "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_" for c in stats_id):
            raise ValueError("Invalid stats_id")
        if len(stats_id) < 11 or len(stats_id) > 32:
            raise ValueError("stats_id has the wrong length")

        REPORTS_DIR.mkdir(exist_ok=True)
        filename = REPORTS_DIR / stats_id

        new_data["timestamp_received_by_bot"] = int(time.time())

        if os.path.exists(filename):
            with open(filename) as file:
                existing_data = json.load(file)
        else:
            existing_data = []
        existing_data.append(new_data)

        with open(filename, "w") as file:
            json.dump(existing_data, file, indent=4)

        bot.logger.info(f"Successfully saved statistics from message {msg.id}.")

        bot.rpc.send_reaction(
            accid,
            msg.id,
            ["❤️"],
        )

    except Exception:
        bot.logger.exception("Could not parse self_reporting message")
        bot.rpc.misc_send_text_message(
            accid,
            chatid,
            "Sorry, I couldn't understand your message.\n\nI am a bot for receiving statistics about your usage of Delta Chat. All other messages will be ignored.",
        )
    finally:
        cleanup_after_message(bot, accid, msg.chat_id)


def cleanup_after_message(bot, accid, chat_id):
    contact_ids = bot.rpc.get_chat_contacts(accid, chat_id)

    # First delete the chat,
    # because contacts that are still in a chat can't be deleted
    bot.rpc.delete_chat(accid, chat_id)
    bot.logger.info(f"Cleaned up chat {chat_id}.")

    for contact_id in contact_ids:
        if contact_id <= SpecialContactId.LAST_SPECIAL:
            continue
        try:
            bot.rpc.delete_contact(accid, contact_id)
            bot.logger.info(f"Cleaned up contact {contact_id}.")
        except Exception:
            bot.logger.exception("Could not delete contact")


@cli.on(events.RawEvent)
def log_event(bot, accid, event):
    if event.kind == EventType.INFO:
        bot.logger.info(event.msg)
    elif event.kind == EventType.WARNING:
        bot.logger.warning(event.msg)
    elif event.kind == EventType.ERROR:
        bot.logger.error(event.msg)
    else:
        bot.logger.info(f"Event: {event}")


@cli.on_init
def on_init(bot, args):
    bot.logger.info("Initializing CLI with args: %s", args)
    for accid in bot.rpc.get_all_account_ids():
        bot.logger.info("Using settings: %s", bot.rpc.get_info(accid))
        bot.rpc.set_config(accid, "delete_server_after", "1")
        bot.rpc.set_config(accid, "delete_device_after", "3600")


def main():
    cli.start()


if __name__ == "__main__":
    main()
```

I wrote tests for it with help from AI. Concretely, I asked AI to generate tests, then reviewed them, iteratively asking the AI for changes and making changes myself.

This is the helper file (`tests/conftest.py`):

```python
import json
import logging
import threading
import time
import pytest
from pathlib import Path
from deltachat2 import Bot, events
from deltachat2.rpc import Rpc
from deltachat2.transport import IOTransport
from deltachat2.types import EventType

CHATMAIL_QR = "dcaccount:ci-chatmail.testrun.org"


def make_rpc(accounts_dir: Path) -> tuple:
    transport = IOTransport(accounts_dir=str(accounts_dir))
    transport.start()
    return Rpc(transport), transport


def wait_for_event(rpc, kind, predicate=lambda e: True, timeout=30):
    """Poll get_next_event until an event matching kind and predicate arrives."""
    deadline = time.time() + timeout
    while time.time() < deadline:
        event = rpc.get_next_event()
        if event.event.kind == kind and predicate(event.event):
            return event
    raise TimeoutError(f"Timed out waiting for event {kind}")


@pytest.fixture(scope="session")
def bot_rpc(tmp_path_factory):
    accounts_dir = tmp_path_factory.mktemp("bot_accounts")
    rpc, transport = make_rpc(accounts_dir)
    yield rpc
    transport.close()


@pytest.fixture(scope="session")
def user_rpc(tmp_path_factory):
    accounts_dir = tmp_path_factory.mktemp("user_accounts")
    rpc, transport = make_rpc(accounts_dir)
    yield rpc
    transport.close()


@pytest.fixture(scope="session")
def bot_accid(bot_rpc):
    accid = bot_rpc.add_account()
    bot_rpc.set_config(accid, "displayname", "TestBot")
    bot_rpc.set_config(accid, "bot", "1")
    bot_rpc.add_transport_from_qr(accid, CHATMAIL_QR)
    return accid


@pytest.fixture(scope="session")
def user_accid(user_rpc):
    accid = user_rpc.add_account()
    user_rpc.set_config(accid, "displayname", "TestUser")
    user_rpc.add_transport_from_qr(accid, CHATMAIL_QR)
    return accid


@pytest.fixture(scope="session")
def reports_dir(tmp_path_factory):
    return tmp_path_factory.mktemp("reports")


@pytest.fixture(scope="session", autouse=True)
def running_bot(bot_rpc, bot_accid, reports_dir):
    import self_reporting_bot
    self_reporting_bot.REPORTS_DIR = reports_dir

    from self_reporting_bot import cli
    bot = Bot(bot_rpc, cli._hooks)

    t = threading.Thread(target=bot.run_forever, args=(bot_accid,), daemon=True)
    t.start()
    yield bot


@pytest.fixture(scope="session")
def chat_id_with_bot(bot_rpc, bot_accid, user_rpc, user_accid):
    """A chat between user and bot with keys already exchanged."""

    qr = bot_rpc.get_chat_securejoin_qr_code(bot_accid, None)

    chat_id = user_rpc.secure_join(user_accid, qr)

    wait_for_event(
        user_rpc,
        EventType.SECUREJOIN_JOINER_PROGRESS,
        predicate=lambda e: e.progress == 1000,
    )

    return chat_id
```

The tests (`tests/test_integration.py`):

```python
# Run with `pytest tests` (you need to `pip install pytest` in a virtualenv before)

import json
import time
import pytest
from pathlib import Path


def wait_for(condition, timeout=5, interval=0.3, msg="condition never met"):
    """Poll until condition() is truthy or timeout."""
    deadline = time.time() + timeout
    while time.time() < deadline:
        result = condition()
        if result:
            return result
        time.sleep(interval)
    raise TimeoutError(msg)


def send_statistics(rpc, accid, chat_id, stats_data, tmp_path, filename="statistics.txt"):
    stats_file = tmp_path / filename
    stats_file.write_text(json.dumps(stats_data))
    from deltachat2.types import MsgData
    return rpc.send_msg(accid, chat_id, MsgData(file=str(stats_file)))


def assert_bot_cleaned_up(bot_rpc, bot_accid):
    chat_ids = bot_rpc.get_chatlist_entries(bot_accid, None, None, None)
    assert len(chat_ids) == 0

    contacts = bot_rpc.get_contacts(bot_accid, 0, None)
    assert len(contacts) == 0


def test_valid_report_gets_reaction(
    user_rpc, user_accid, bot_rpc, bot_accid, chat_id_with_bot, reports_dir, tmp_path
):
    stats = {"stats_id": "test_reaction_id", "metric": 1}
    sent_stats_msg_id = send_statistics(user_rpc, user_accid, chat_id_with_bot, stats, tmp_path)

    def got_reaction():
        reactions = user_rpc.get_message_reactions(user_accid, sent_stats_msg_id)
        return reactions is not None and reactions.reactions[0].emoji == '❤️'

    wait_for(got_reaction, msg="Bot never sent a reaction")

    assert_bot_cleaned_up(bot_rpc, bot_accid)


def test_valid_report_is_saved_to_disk(
    chat_id_with_bot, user_rpc, bot_rpc, bot_accid, user_accid, reports_dir, tmp_path
):
    stats_id = "test_saved_to_disk_id"
    stats = {"stats_id": stats_id, "some_metric": 99}
    send_statistics(user_rpc, user_accid, chat_id_with_bot, stats, tmp_path)

    def report_exists():
        p = reports_dir / stats_id
        try:
            return p.exists() and json.loads(p.read_text())
        except (json.JSONDecodeError, OSError):
            return False

    saved = wait_for(report_exists, msg="Report never appeared on disk")
    assert saved[0]["some_metric"] == 99
    assert "timestamp_received_by_bot" in saved[0]

    assert_bot_cleaned_up(bot_rpc, bot_accid)


def test_invalid_stats_id_sends_error_message(
    chat_id_with_bot, user_rpc, bot_rpc, bot_accid, user_accid, reports_dir, tmp_path
):
    stats = {"stats_id": "bad id"}  # spaces are invalid
    send_statistics(user_rpc, user_accid, chat_id_with_bot, stats, tmp_path)

    def got_error_reply():
        msg_ids = user_rpc.get_message_ids(user_accid, chat_id_with_bot, False, False)
        messages = user_rpc.get_messages(user_accid, msg_ids)
        return any(
            "couldn't understand" in (getattr(m, "text", "") or "")
            for m in messages.values()
        )

    wait_for(got_error_reply, msg="Bot never sent error message")
    time.sleep(1)
    assert not (reports_dir / "bad id").exists()

    assert_bot_cleaned_up(bot_rpc, bot_accid)
```

## Evaluation scripts for the quality of the data collected in the pre-study

In order to assess whether the collected data from the pre-study could be used to address RQ1 and RQ2, I extended evaluation scripts that the researchers had written. To be precise, at the time of writing the evaluation section of my thesis, the researchers had code to load and filter the data, but not yet to actually write graphs concerning RQ1 and RQ2. This is the final evaluation script:

```python
# %% Pre-study evaluation script
# # 2026-delta-chat
# 

# %% [markdown]
# ## Imports

# %%
from pathlib import Path
import json
import pandas as pd
import numpy as np
import seaborn as sns
import matplotlib.pyplot as plt
import matplotlib.ticker as ticker

DATA_PATH = Path("..")

DC_TELEMETRY_REPORTS_DIR = Path(DATA_PATH/'reports')

# %% [markdown]
# ## Load Delta Chat Telemetry data
# 
# The data is stored in individual .json files, as specified in `stats_documentation.md`

# %%
# Only count reports with sending enabled after 2026-05-01 00:00:00 UTC
START_TIMESTAMP = 1777593600

def was_sending_enabled_after(report):
    # Uncomment the following line to use the full dataset;
    # leave it commented out it to just use the prestudy dataset
    #return True

    enabled_ts = report.get("sending_enabled_timestamps", [])
    
    if not isinstance(enabled_ts, list) or len(enabled_ts) == 0:
        return False
    
    return enabled_ts[0] > START_TIMESTAMP

def load_df_telemetry_reports(dir):
    dfs = []
    loaded = 0
    total = 0

    for file in dir.iterdir():
        if file.is_file:
            total += 1
            with open(file) as data_file:
                data = json.load(data_file)
                last_report = data[-1]
                if was_sending_enabled_after(last_report):
                    loaded += 1
                    temp_df = pd.json_normalize(data)
                    dfs.append(temp_df)

    print(f"Loaded {loaded} out of {total} telemetry reports")
    return pd.concat(dfs, ignore_index=True)

dc_raw_data =load_df_telemetry_reports(DC_TELEMETRY_REPORTS_DIR)


# %% [markdown]
# ### Anonymization
# 
# We need to anonymize the key age, as the exact creation timestamp might be an identifier. We do this by bining the creation timestamps.

# %%
def bucket_and_replace_with_mean(df, col):
    df_new = df.copy()

    # 1. Extract only valid lists (skip NaN / non-list)
    valid_mask = df_new[col].apply(lambda x: isinstance(x, (list, np.ndarray)))

    all_vals = np.concatenate(df_new.loc[valid_mask, col].values) \
        if valid_mask.any() else np.array([])

    if len(all_vals) == 0:
        return df_new  # nothing to bucket

    # 2. Helper: check validity of k bins
    def valid_k(k):
        bins = np.linspace(all_vals.min(), all_vals.max(), k + 1)
        bin_ids = np.digitize(all_vals, bins, right=False) - 1
        bin_ids = np.clip(bin_ids, 0, k - 1)

        counts = np.bincount(bin_ids, minlength=k)
        return np.all((counts == 0) | (counts >= 2))

    # 3. Find best number of bins
    max_k = max(1, len(all_vals) // 2)
    best_k = 1

    for k in range(max_k, 0, -1):
        if valid_k(k):
            best_k = k
            break

    print(f"Creating {best_k} bins.")

    # 4. Build final bins
    bins = np.linspace(all_vals.min(), all_vals.max(), best_k + 1)

    bin_ids = np.digitize(all_vals, bins, right=False) - 1
    bin_ids = np.clip(bin_ids, 0, best_k - 1)

    # 5. Compute means per bin
    bin_means = {
        b: int(np.rint(all_vals[bin_ids == b].mean()))
        for b in range(best_k)
        if np.any(bin_ids == b)
    }

    # 6. Replace values in each list, safely handling NaN rows
    def replace_list(x):
        if not isinstance(x, (list, np.ndarray)):
            return x  # keep NaN / None / scalars unchanged

        arr = np.array(x)
        ids = np.digitize(arr, bins, right=False) - 1
        ids = np.clip(ids, 0, best_k - 1)
        return [bin_means[i] for i in ids]

    df_new[col] = df_new[col].apply(replace_list)

    return df_new
dc_anonymized = bucket_and_replace_with_mean(dc_raw_data, 'key_create_timestamps')

# Diff between the anonymized and raw timestamps
diff = dc_raw_data['key_create_timestamps'].str[0] - dc_anonymized['key_create_timestamps'].str[0]
diff.describe()

# %%
dc_raw_data = dc_anonymized
dc_raw_data.head()

# %% [markdown]
# ### Enriching with Aggregate Columns
# 
# We add several aggregate columns on number of keys, telemetry enablement, contacts, and messages.

# %%
dc_raw_data['keys_total'] = dc_raw_data["key_create_timestamps"].apply(lambda x: len(x) if isinstance(x, list) else 0)

dc_raw_data['sending_enabled_timestamps.Total'] = dc_raw_data["sending_enabled_timestamps"].apply(lambda x: len(x) if isinstance(x, list) else 0)

dc_raw_data['contact_stats.Total'] = dc_raw_data['contact_stats'].apply(lambda x: len(x) if isinstance(x, list) else 0)

dc_raw_data['message_stats.Total'] = dc_raw_data[['message_stats.Single.verified', 'message_stats.Single.unverified_encrypted', 'message_stats.Single.unencrypted', 'message_stats.Single.only_to_self', 'message_stats.Group.verified', 'message_stats.Group.unverified_encrypted', 'message_stats.Group.unencrypted', 'message_stats.Group.only_to_self', 'message_stats.OutBroadcast.verified', 'message_stats.OutBroadcast.unverified_encrypted', 'message_stats.OutBroadcast.unencrypted', 'message_stats.OutBroadcast.only_to_self']].sum(axis=1)

# %% [markdown]
# ### Filtering
# 
# We filter out data that does not belong to our study

# %%
print(f"Raw data: {len(dc_raw_data)}")

# remove data collected before Nov 8 (when the telemetry collection for this study started in deltachat)
dc_filtered_data = dc_raw_data.loc[dc_raw_data['sending_enabled_timestamps'].str[0] > 1762642799]
print(f"After date filter: {len(dc_filtered_data)}")

# remove data with core version < 2.23
valid_versions = ['2.23.0','2.24.0','2.25.0','2.26.0','2.27.0','2.28.0','2.29.0','2.30.0','2.31.0','2.32.0','2.33.0','2.34.0','2.35.0','2.36.0','2.37.0','2.38.0','2.39.0','2.40.0','2.41.0','2.42.0','2.43.0','2.44.0','2.45.0','2.46.0','2.47.0','2.48.0','2.49.0','2.50.0','2.51.0']
dc_filtered_data = dc_filtered_data[(dc_filtered_data['core_version'].isin(valid_versions))]
print(f"After version filter: {len(dc_filtered_data)}")

# take only latest measurement
dc_latest_filtered_data = dc_filtered_data.loc[dc_filtered_data.groupby('stats_id')['timestamp_received_by_bot'].idxmax()]
print(f"After taking latest per user: {len(dc_latest_filtered_data)}")

# remove measurements with 0 contacts (Those have never used Delta Chat)
dc_latest_filtered_data = dc_latest_filtered_data.loc[dc_latest_filtered_data['contact_stats.Total'] > 0]
print(f"After removing 0-contact users: {len(dc_latest_filtered_data)}")

# assert that each telemetry report contains exactly one key_created_timestamp
assert (dc_latest_filtered_data["keys_total"]==1).all()
# ensure that we do not have any old data without sending_enabled_timestamps (which existed before our study was conducted)
assert (dc_latest_filtered_data['sending_enabled_timestamps.Total']>=1).all()

# print number of valid measurement participants
print(f"Number of telemetry reports: {len(dc_latest_filtered_data['stats_id'])}")
dc_latest_filtered_data.head()

# %% [markdown]
# First received telemetry data

# %%
dc_filtered_data['timestamp_received_by_bot'].min()

# %% [markdown]
# ### Age of the keys
# 
# By year:

# %%
dc_key_creation = dc_latest_filtered_data["key_create_timestamps"].apply(lambda x: x[0])

dc_key_creation_years = pd.to_datetime(
    dc_key_creation, 
    unit='s'
).dt.year.value_counts().sort_index()

ax = dc_key_creation_years.plot(kind='bar', figsize=(6, 4))

plt.xticks(rotation=45, ha="right")
plt.show()

dc_key_creation_years

# %% [markdown]
# By bin (each identified by a UNIX timestamp):

# %%
dc_key_creation.value_counts().sort_index().plot(kind="bar", figsize=(24, 6))
plt.xticks(rotation=45, ha="right")
plt.show()

# %% [markdown]
# ### RQ1: How prevalent is key verification among Delta Chat users?

# %%
import matplotlib.ticker as mtick

def expand_contacts(df):
    """Return a flat DataFrame with one row per contact, keeping is_chatmail."""
    rows = []
    for _, report in df.iterrows():
        contacts = report.get("contact_stats")
        if not isinstance(contacts, list):
            continue
        for c in contacts:
            rows.append({
                "is_chatmail":   report["is_chatmail"],
                "verified":      c["verified"],
                "bot":           c.get("bot", False),
                "transitive_chain": c.get("transitive_chain"),
            })
    return pd.DataFrame(rows)

contacts_df = expand_contacts(dc_latest_filtered_data)

# Collapse TransitiveViaBot into its own bucket; keep Direct / Transitive / Opportunistic / Unencrypted
VERIFIED_ORDER = ["Direct", "Transitive", "TransitiveViaBot", "Opportunistic", "Unencrypted"]
VERIFIED_LABELS = {
    "Direct":           "Directly verified",
    "Transitive":       "Transitively verified",
    "TransitiveViaBot": "Transitively verified\nvia a bot",
    "Opportunistic":    "Encrypted, but unverified",
    "Unencrypted":      "Unencrypted",
}
COLORS = {
    "Direct":          "#2ecc71",
    "Transitive":      "#27ae60",
    "TransitiveViaBot":"#f39c12",
    "Opportunistic":   "#3498db",
    "Unencrypted":     "#e74c3c",
}

# ── plot ──────────────────────────────────────────────────────────────────────

fig, axes = plt.subplots(1, 2, figsize=(12, 5), sharey=False)
fig.suptitle("RQ1 - Fig 1: Contact Verification Status", fontsize=14, fontweight="bold")

for ax, (chatmail_val, label) in zip(axes, [(True, "Chatmail"), (False, "Classical e-mail")]):
    subset = contacts_df[contacts_df["is_chatmail"] == chatmail_val]
    counts = subset["verified"].value_counts().reindex(VERIFIED_ORDER, fill_value=0)
    total  = counts.sum()
    pcts   = counts / total * 100

    bars = ax.bar(
        counts.index,
        pcts,
        color=[COLORS[v] for v in counts.index],
        edgecolor="white", linewidth=0.8,
    )
    ax.set_title(f"{label}\n(n = {total:,} contacts)", fontsize=11)
    ax.set_ylabel("Share of contacts (%)")
    ax.yaxis.set_major_formatter(mtick.PercentFormatter())
    ax.set_ylim(0, 100)
    ax.set_xticks(range(len(VERIFIED_ORDER)))
    ax.set_xticklabels([VERIFIED_LABELS[v] for v in VERIFIED_ORDER], rotation=30, ha="right")
    for bar, pct in zip(bars, pcts):
        if pct > 1:
            ax.text(bar.get_x() + bar.get_width() / 2, bar.get_height() + 0.8,
                    f"{pct:.1f}%", ha="center", va="bottom", fontsize=8)

plt.tight_layout()
plt.show()

# %%
fig, axes = plt.subplots(1, 2, figsize=(12, 5), sharey=False)
fig.suptitle("RQ1 - Contact Verification Status (human contacts only, bots excluded)",
             fontsize=13, fontweight="bold")

for ax, (chatmail_val, label) in zip(axes, [(True, "Chatmail"), (False, "Classical e-mail")]):
    subset = contacts_df[
        (contacts_df["is_chatmail"] == chatmail_val) &
        (~contacts_df["bot"])
    ]
    counts = subset["verified"].value_counts().reindex(VERIFIED_ORDER, fill_value=0)
    total  = counts.sum()
    pcts   = counts / total * 100

    bars = ax.bar(
        counts.index,
        pcts,
        color=[COLORS[v] for v in counts.index],
        edgecolor="white", linewidth=0.8,
    )
    ax.set_title(f"{label}\n(n = {total:,} human contacts)", fontsize=11)
    ax.set_ylabel("Share of contacts (%)")
    ax.yaxis.set_major_formatter(mtick.PercentFormatter())
    ax.set_ylim(0, 100)
    ax.set_xticks(range(len(VERIFIED_ORDER)))
    ax.set_xticklabels([VERIFIED_LABELS[v] for v in VERIFIED_ORDER], rotation=30, ha="right")
    for bar, pct in zip(bars, pcts):
        ax.text(bar.get_x() + bar.get_width() / 2, bar.get_height() + 0.8,
                f"{pct:.1f}%", ha="center", va="bottom", fontsize=8)

plt.tight_layout()
plt.savefig("rq1_fig2_contact_verification_no_bots.png", dpi=150)
plt.show()

# %%
MSG_COLS = {
    "1:1 chats":       ["message_stats.Single.verified",
                     "message_stats.Single.unverified_encrypted",
                     "message_stats.Single.unencrypted",
                     "message_stats.Single.only_to_self"],
    "Groups":        ["message_stats.Group.verified",
                     "message_stats.Group.unverified_encrypted",
                     "message_stats.Group.unencrypted",
                     "message_stats.Group.only_to_self"],
}
ENC_LABELS   = ["verified", "unverified_encrypted", "unencrypted", "only_to_self"]
ENC_COLORS   = ["#2ecc71", "#3498db", "#e74c3c", "#95a5a6"]
ENC_DISPLAY  = ["Verified", "Encrypted, but unverified", "Unencrypted", "Only to self"]
ENC_HATCHES = ["", "///", "xxx", "..."]

fig, axes = plt.subplots(1, 2, figsize=(13, 5), sharey=False)
fig.suptitle("RQ1 - Outgoing Messages by Encryption Level and Chat Type",
             fontsize=13, fontweight="bold")

for ax, (chatmail_val, label) in zip(axes, [(True, "Chatmail"), (False, "Classical e-mail")]):
    subset = dc_latest_filtered_data[dc_latest_filtered_data["is_chatmail"] == chatmail_val]
    totals_by_type = {}
    for chat_type, cols in MSG_COLS.items():
        # only sum columns that actually exist in the df
        existing = [c for c in cols if c in subset.columns]
        sums = subset[existing].sum()
        # re-index to always have all four buckets
        sums.index = [c.split(".")[-1] for c in sums.index]
        sums = sums.reindex(ENC_LABELS, fill_value=0)
        totals_by_type[chat_type] = sums

    x      = np.arange(len(MSG_COLS))
    width  = 0.6
    bottom = np.zeros(len(MSG_COLS))
    for enc, color, disp, hatch in zip(ENC_LABELS, ENC_COLORS, ENC_DISPLAY, ENC_HATCHES):
        heights = [totals_by_type[t][enc] for t in MSG_COLS]
        ax.bar(x, heights, width, bottom=bottom, label=disp, color=color,
            edgecolor="white", hatch=hatch)
        bottom += np.array(heights)

    ax.set_title(f"{label}", fontsize=11)
    ax.set_xticks(x)
    ax.set_xticklabels(list(MSG_COLS.keys()))
    ax.set_ylabel("Message count (sum across users)")
    ax.yaxis.set_major_formatter(mtick.FuncFormatter(lambda v, _: f"{int(v):,}"))
    if ax == axes[0]:
        ax.legend(loc="upper right", fontsize=11)

plt.tight_layout()
plt.show()

broadcast_cols = [c for c in dc_latest_filtered_data.columns if "OutBroadcast" in c]
print(dc_latest_filtered_data[broadcast_cols].sum())

# %%
# ── RQ2: How do Delta Chat users perform key verification? ────────────────────

# ── 1. Securejoin invites: active vs. incidental ──────────────────────────────

invites = dc_latest_filtered_data["securejoin_invites"].explode().dropna()
invites_df = pd.json_normalize(invites)

# Incidental = new contact OR group/broadcast invite; filter already-verified
invites_df = invites_df[~invites_df["already_verified"].fillna(False)]
invites_df["kind"] = np.where(
    (~invites_df["already_existed"].fillna(False)) | (invites_df["typ"] != "contact"),
    "Incidental", "Active"
)

fig, axes = plt.subplots(1, 2, figsize=(13, 4))
fig.suptitle("RQ2 - Securejoin Invite Characteristics", fontweight="bold")

# Active vs. incidental
invites_df["kind"].value_counts().plot.bar(ax=axes[0], color=["#3498db","#2ecc71"], edgecolor="white", rot=0)
axes[0].set_title("Active vs. Incidental")
axes[0].set_ylabel("Count"); axes[0].set_xlabel("")

# Invite type
TYPE_LABELS = {
    "contact":   "1:1 Invite",
    "group":     "Group Invite",
    "broadcast": "Broadcast Channel Invite",
}
invites_df["typ"] = invites_df["typ"].map(TYPE_LABELS)
invites_df["typ"].value_counts().plot.bar(ax=axes[1], color=["#9b59b6","#e67e22","#1abc9c"], edgecolor="white", rot=0)
axes[1].set_title("Invite Type")
axes[1].set_xlabel("")

# ── 2. Invite source (out-of-band channel security) ──────────────────────────

source_cols = [c for c in dc_latest_filtered_data.columns if c.startswith("securejoin_sources.")]
sources = dc_latest_filtered_data[source_cols].sum().rename(lambda c: c.split(".")[-1]).sort_values(ascending=True)

fig, ax = plt.subplots(figsize=(7, 4))
fig.suptitle("RQ2 – Fig 2: Invite Code Origin (out-of-band channel)", fontweight="bold")
sources.plot.barh(ax=ax, color="#3498db", edgecolor="white")
ax.set_xlabel("Total count"); ax.set_ylabel("")
for bar, val in zip(ax.patches, sources):
    ax.text(bar.get_width() + sources.max() * 0.01, bar.get_y() + bar.get_height() / 2,
            f"{int(val):,}", va="center", fontsize=9)
plt.tight_layout()
plt.savefig("rq2_fig2_invite_sources.png", dpi=150); plt.show()

# ── 3. UI path used to initiate QR scan ──────────────────────────────────────

uipath_cols = [c for c in dc_latest_filtered_data.columns if c.startswith("securejoin_uipaths.")]
uipaths = dc_latest_filtered_data[uipath_cols].sum().rename(lambda c: c.split(".")[-1])

fig, ax = plt.subplots(figsize=(5, 3))
fig.suptitle("RQ2 - UI Path to QR Scan", fontweight="bold")
uipaths.plot.bar(ax=ax, color=["#9b59b6","#e67e22","#1abc9c"], edgecolor="white", rot=0)
ax.set_ylabel("Count"); ax.set_xlabel("")
plt.tight_layout()
plt.savefig("rq2_fig3_ui_paths.png", dpi=150); plt.show()

# ── 4. Transitive verification chains: length & bot involvement ───────────────

transitive = contacts_df[contacts_df["verified"].isin(["Transitive", "TransitiveViaBot"])].copy()
transitive["chain_len"] = transitive["transitive_chain"].clip(upper=10)  # cap display at 10 (100=loop)
transitive["has_bot"] = transitive["verified"] == "TransitiveViaBot"

fig, axes = plt.subplots(1, 2, figsize=(11, 4))
fig.suptitle("RQ2 – Fig 4: Transitive Verification Chains", fontweight="bold")

# Chain length distribution, split by bot involvement
for has_bot, label, color in [(False, "No bot in chain", "#27ae60"), (True, "Bot in chain", "#e74c3c")]:
    sub = transitive[transitive["has_bot"] == has_bot]["chain_len"].value_counts().sort_index()
    axes[0].bar(sub.index + (0.2 if has_bot else -0.2), sub.values, width=0.38,
                label=label, color=color, edgecolor="white")
axes[0].set_title("Chain Length Distribution")
axes[0].set_xlabel("Chain length (capped at 10)"); axes[0].set_ylabel("Count")
axes[0].legend(); axes[0].set_xticks(range(1, 11))

# Bot vs. no-bot share (pie)
bot_counts = transitive["has_bot"].value_counts()
axes[1].pie(bot_counts, labels=["No bot in chain","Bot in chain"], colors=["#27ae60","#e74c3c"],
            autopct="%1.1f%%", startangle=90, wedgeprops={"edgecolor":"white"})
axes[1].set_title("Bot Involvement in Transitive Chains")

plt.tight_layout()
plt.savefig("rq2_fig4_transitive_chains.png", dpi=150); plt.show()

# %%
invites_df.head(50)

# %%
# NOTE: This is just some AI-generated code.
# I created it mainly out of interest, but I am
# not familiar enough with statistics to vouch for the quality of the code.
# It produces plausible results, but this is all I can say.
# I did not use it in the bachelor thesis.

from scipy import stats
import statsmodels.api as sm
from datetime import timezone

# ── Shared preparation ────────────────────────────────────────────────────────

df = dc_latest_filtered_data.copy()
def verified_message_share(row):
    total_cols = [
        "message_stats.Single.verified", "message_stats.Single.unverified_encrypted",
        "message_stats.Single.unencrypted", "message_stats.Single.only_to_self",
        "message_stats.Group.verified", "message_stats.Group.unverified_encrypted",
        "message_stats.Group.unencrypted", "message_stats.Group.only_to_self",
        "message_stats.OutBroadcast.verified", "message_stats.OutBroadcast.unverified_encrypted",
        "message_stats.OutBroadcast.unencrypted", "message_stats.OutBroadcast.only_to_self",
    ]
    verified_cols = [c for c in total_cols if c.endswith(".verified")]
    total    = sum(row.get(c, 0) or 0 for c in total_cols if c in row.index)
    verified = sum(row.get(c, 0) or 0 for c in verified_cols if c in row.index)
    return verified / total if total > 0 else np.nan

df["verified_msg_share"] = df.apply(verified_message_share, axis=1)

# Key age in days (from first key creation timestamp to report receipt)
df["key_age_days"] = (df["timestamp_received_by_bot"] - df["key_create_timestamps"].apply(lambda x: x[0])) / 86400

# Verified contact share per user (reuse logic from RQ1)
def contact_verified_share(contacts, bot_filter=None, new_filter=None):
    """Returns (verified_count, total_count) for contacts matching filters."""
    if not isinstance(contacts, list):
        return np.nan, 0
    filtered = contacts
    if bot_filter is not None:
        filtered = [c for c in filtered if c.get("bot", False) == bot_filter]
    if new_filter is not None:
        filtered = [c for c in filtered if c.get("new", False) == new_filter]
    total = len(filtered)
    if total == 0:
        return np.nan, 0
    verified = sum(1 for c in filtered if c.get("verified") in ("Direct", "Transitive", "TransitiveViaBot", "Opportunistic"))
    return verified / total, total

df["verified_contact_share"], df["n_contacts"] = zip(
    *df["contact_stats"].apply(lambda x: contact_verified_share(x, bot_filter=False))
)

# Number of chats = contacts with a direct_chat (proxy for chat count)
df["n_direct_chats"] = df["contact_stats"].apply(
    lambda x: sum(1 for c in x if c.get("direct_chat", False)) if isinstance(x, list) else 0
)

# Messages per week
df["study_duration_weeks"] = (
    df["timestamp_received_by_bot"] - df["sending_enabled_timestamps"].apply(lambda x: x[0])
) / (86400 * 7)
df["msgs_per_week"] = df["message_stats.Total"] / df["study_duration_weeks"].clip(lower=0.01)

# Drop users with no contacts or no messages
df_ha_hb = df.dropna(subset=["verified_contact_share", "key_age_days"]).copy()
df_hf    = df[df["message_stats.Total"] > 0].dropna(subset=["verified_msg_share"]).copy()

print(f"Sample sizes: HA/HB n={len(df_ha_hb)}, HF n={len(df_hf)}")

# ── HA: Key age → verified contact share (directed: older key = lower share) ──

X_ha = sm.add_constant(df_ha_hb["key_age_days"])
model_ha = sm.OLS(df_ha_hb["verified_contact_share"], X_ha).fit()

slope_ha = model_ha.params["key_age_days"]
# One-tailed p-value: we hypothesize slope < 0
p_ha_onetailed = model_ha.pvalues["key_age_days"] / 2 if slope_ha < 0 else 1.0

print("\n── HA: Key age → share of verified contacts (directed, H1: slope < 0) ──")
print(f"  Slope:        {slope_ha:.6f} (share per day)")
print(f"  R²:           {model_ha.rsquared:.4f}")
print(f"  p (2-tailed): {model_ha.pvalues['key_age_days']:.4f}")
print(f"  p (1-tailed): {p_ha_onetailed:.4f}")
print(f"  → {'Reject H0' if p_ha_onetailed < 0.05 else 'Fail to reject H0'} at α=0.05")

# ── HB: Number of chats → verified contact share (undirected) ────────────────

X_hb = sm.add_constant(df_ha_hb["n_direct_chats"])
model_hb = sm.OLS(df_ha_hb["verified_contact_share"], X_hb).fit()

print("\n── HB: Number of chats → share of verified contacts (undirected) ──")
print(f"  Slope:        {model_hb.params['n_direct_chats']:.6f}")
print(f"  R²:           {model_hb.rsquared:.4f}")
print(f"  p (2-tailed): {model_hb.pvalues['n_direct_chats']:.4f}")
print(f"  → {'Reject H0' if model_hb.pvalues['n_direct_chats'] < 0.05 else 'Fail to reject H0'} at α=0.05")

# ── HF: Messages/week → share of verified messages (directed: more = higher) ─

X_hf = sm.add_constant(df_hf["msgs_per_week"])
model_hf = sm.OLS(df_hf["verified_msg_share"], X_hf).fit()

slope_hf = model_hf.params["msgs_per_week"]
p_hf_onetailed = model_hf.pvalues["msgs_per_week"] / 2 if slope_hf > 0 else 1.0

print("\n── HF: Messages/week → share of verified messages (directed, H1: slope > 0) ──")
print(f"  Slope:        {slope_hf:.6f}")
print(f"  R²:           {model_hf.rsquared:.4f}")
print(f"  p (2-tailed): {model_hf.pvalues['msgs_per_week']:.4f}")
print(f"  p (1-tailed): {p_hf_onetailed:.4f}")
print(f"  → {'Reject H0' if p_hf_onetailed < 0.05 else 'Fail to reject H0'} at α=0.05")

# ── HG: Verified share of new vs. existing contacts (paired per user) ─────────

def split_contact_shares(contacts):
    """Returns (share_new, share_existing) for non-bot contacts."""
    if not isinstance(contacts, list):
        return np.nan, np.nan
    human = [c for c in contacts if not c.get("bot", False)]
    new_contacts = [c for c in human if c.get("new", False)]
    old_contacts = [c for c in human if not c.get("new", False)]
    def share(lst):
        if not lst:
            return np.nan
        return sum(1 for c in lst if c.get("verified") in ("Direct", "Transitive", "TransitiveViaBot", "Opportunistic")) / len(lst)
    return share(new_contacts), share(old_contacts)

df["share_new"], df["share_existing"] = zip(*df["contact_stats"].apply(split_contact_shares))

# Only users who have both new and existing contacts
df_hg = df.dropna(subset=["share_new", "share_existing"]).copy()

t_stat, p_hg_twotailed = stats.ttest_rel(df_hg["share_new"], df_hg["share_existing"])
# Directed: new > existing, so one-tailed
p_hg_onetailed = p_hg_twotailed / 2 if t_stat > 0 else 1.0

print("\n── HG: Verified share — new vs. existing contacts (paired t-test, H1: new > existing) ──")
print(f"  n (users with both):  {len(df_hg)}")
print(f"  Mean share (new):     {df_hg['share_new'].mean():.4f}")
print(f"  Mean share (existing):{df_hg['share_existing'].mean():.4f}")
print(f"  t-statistic:          {t_stat:.4f}")
print(f"  p (2-tailed):         {p_hg_twotailed:.4f}")
print(f"  p (1-tailed):         {p_hg_onetailed:.4f}")
print(f"  → {'Reject H0' if p_hg_onetailed < 0.05 else 'Fail to reject H0'} at α=0.05")

# %%
one_week_ago = dc_latest_filtered_data["timestamp_received_by_bot"].max() \
  - 7 * 24 * 3600
dc_latest_filtered_data2 = dc_latest_filtered_data.loc[
  dc_latest_filtered_data["sending_enabled_timestamps"].str[0] < one_week_ago
]
abandoned = dc_latest_filtered_data2["timestamp_received_by_bot"] < one_week_ago
print(f"Last report received longer than one week ago: {abandoned.sum()} \
  / {len(dc_latest_filtered_data2)} ({abandoned.mean() * 100:.1f}%)")

last_received = pd.to_datetime(dc_latest_filtered_data2["timestamp_received_by_bot"], unit="s")
start = last_received.min().floor("D")
end = last_received.max().ceil("D")
bins = pd.date_range(start, end, freq="D")

last_received.hist(bins=bins, figsize=(8, 4))
plt.ylabel("Number of users")
plt.title("When was the last report received?")
plt.xticks(rotation=45, ha="right")
plt.tight_layout()
plt.show()

eleven_days_ago = dc_latest_filtered_data2["timestamp_received_by_bot"].max() - 11 * 24 * 3600
dc_latest_filtered_data2 = dc_latest_filtered_data.loc[
  dc_latest_filtered_data["sending_enabled_timestamps"].str[0] < eleven_days_ago
]
abandoned = dc_latest_filtered_data2["timestamp_received_by_bot"] < eleven_days_ago
print(f"Last report received longer than 11 days ago: {abandoned.sum()} / {len(dc_latest_filtered_data2)} ({abandoned.mean() * 100:.1f}%)")
```

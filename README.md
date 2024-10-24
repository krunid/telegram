# telegram

// ตั้งค่า Telegram Bot Token และ Chat ID
const TELEGRAM_BOT_TOKEN = '8xxxx';
const TELEGRAM_CHAT_ID = '4xxxx'; //chat id ตัวเอง
const SHEET_NAME = 'Sheet1';

// ฟังก์ชั่นแปลงวันที่เป็นภาษาไทยแบบเต็ม
function getThaiDate() {
  const date = new Date();
  const thaiDay = ['อาทิตย์','จันทร์','อังคาร','พุธ','พฤหัสบดี','ศุกร์','เสาร์'];
  const thaiMonth = [
    'มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน', 'พฤษภาคม', 'มิถุนายน',
    'กรกฎาคม', 'สิงหาคม', 'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม'
  ];
  
  const day = thaiDay[date.getDay()];
  const d = date.getDate();
  const month = thaiMonth[date.getMonth()];
  const year = date.getFullYear() + 543;
  
  return `📅 วัน${day}ที่ ${d} ${month} พ.ศ. ${year}`;
}

// ฟังก์ชั่นสำหรับสุ่มเลือกค่าจากอาร์เรย์
function getRandomItem(arr) {
  if (!arr || arr.length === 0) return null;
  return arr[Math.floor(Math.random() * arr.length)];
}

// ฟังก์ชั่นสำหรับกรองและสุ่มเลือก URL รูปภาพ
function getRandomImageUrl(imageUrls) {
  const validUrls = imageUrls.filter(url => {
    return url && url.trim() !== '' && 
           (url.startsWith('http://') || url.startsWith('https://'));
  });
  return getRandomItem(validUrls);
}

// ฟังก์ชั่นสำหรับส่งรูปภาพพร้อมข้อความไปยัง Telegram
function sendPhotoToTelegram(imageUrl, caption) {
  if (!imageUrl) {
    Logger.log('ไม่พบ URL รูปภาพ');
    return sendToTelegram(caption);
  }

  caption = caption.replace(/\\n/g, '\n');

  const telegramUrl = `https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendPhoto`;
  const options = {
    'method': 'post',
    'contentType': 'application/json',
    'payload': JSON.stringify({
      'chat_id': TELEGRAM_CHAT_ID,
      'photo': imageUrl,
      'caption': caption,
      'parse_mode': 'HTML'
    })
  };
  
  try {
    const response = UrlFetchApp.fetch(telegramUrl, options);
    const responseData = JSON.parse(response.getContentText());
    
    if (responseData.ok) {
      Logger.log('ส่งรูปภาพและข้อความสำเร็จ');
      return true;
    } else {
      Logger.log('การส่งรูปภาพล้มเหลว: ' + responseData.description);
      return sendToTelegram(caption);
    }
  } catch(e) {
    Logger.log('เกิดข้อผิดพลาดในการส่งรูปภาพ: ' + e.toString());
    return sendToTelegram(caption);
  }
}

// ฟังก์ชั่นดึงข้อมูลประจำวัน
function getDailyContent() {
  try {
    const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);
    if (!sheet) {
      Logger.log('ไม่พบ Sheet ที่ระบุ');
      return { 
        message: `${getThaiDate()}\n\nสวัสดีตอนเช้า\nขอให้เป็นวันที่ดี`, 
        imageUrls: [] 
      };
    }

    const data = sheet.getDataRange().getValues();
    const today = new Date().getDay();
    const thaiDay = ['อาทิตย์','จันทร์','อังคาร','พุธ','พฤหัสบดี','ศุกร์','เสาร์'];
    
    for (let row of data) {
      if (row[0] !== '' && row[0] === today) {
        let message = row[1].toString().trim();
        message = `${getThaiDate()}\n\n${message}`;
        
        const imageUrls = row.slice(2).filter(url => url !== '');
        
        return {
          message: message,
          imageUrls: imageUrls
        };
      }
    }
    
    Logger.log('ไม่พบข้อมูลสำหรับวันนี้ ใช้ข้อความเริ่มต้น');
    return {
      message: `${getThaiDate()}\n\nสวัสดีตอนเช้าวัน${thaiDay[today]}\nขอให้เป็นวันที่ดี`,
      imageUrls: []
    };
    
  } catch(e) {
    Logger.log('เกิดข้อผิดพลาดในการดึงข้อมูล: ' + e.toString());
    return { 
      message: `${getThaiDate()}\n\nสวัสดีตอนเช้า`, 
      imageUrls: [] 
    };
  }
}

// ฟังก์ชั่นสำหรับส่งข้อความเท่านั้น
function sendToTelegram(message) {
  if (!message || message.trim() === '') {
    Logger.log('ไม่สามารถส่งข้อความว่างได้');
    return false;
  }

  message = message.replace(/\\n/g, '\n');

  const telegramUrl = `https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage`;
  const options = {
    'method': 'post',
    'contentType': 'application/json',
    'payload': JSON.stringify({
      'chat_id': TELEGRAM_CHAT_ID,
      'text': message,
      'parse_mode': 'HTML'
    })
  };
  
  try {
    const response = UrlFetchApp.fetch(telegramUrl, options);
    const responseData = JSON.parse(response.getContentText());
    return responseData.ok;
  } catch(e) {
    Logger.log('เกิดข้อผิดพลาดในการส่งข้อความ: ' + e.toString());
    return false;
  }
}

// ฟังก์ชั่นหลักสำหรับส่งข้อความและรูปภาพ
function sendMorningGreeting() {
  try {
    const content = getDailyContent();
    
    if (!content.message || content.message.trim() === '') {
      Logger.log('ไม่มีข้อความที่จะส่ง');
      return;
    }

    const randomImageUrl = getRandomImageUrl(content.imageUrls);
    
    if (randomImageUrl) {
      const success = sendPhotoToTelegram(randomImageUrl, content.message);
      if (success) {
        Logger.log('ส่งรูปภาพและข้อความสำเร็จ');
      } else {
        Logger.log('เกิดข้อผิดพลาดในการส่งรูปภาพและข้อความ');
      }
    } else {
      const success = sendToTelegram(content.message);
      if (success) {
        Logger.log('ส่งข้อความสำเร็จ');
      } else {
        Logger.log('เกิดข้อผิดพลาดในการส่งข้อความ');
      }
    }
  } catch(e) {
    Logger.log('เกิดข้อผิดพลาดในฟังก์ชั่นหลัก: ' + e.toString());
  }
}

// ฟังก์ชั่นสำหรับทดสอบการส่งข้อความ
function testSendMessage() {
  sendMorningGreeting();
}

// สร้าง Trigger อัตโนมัติ
function createTrigger() {
  try {
    const triggers = ScriptApp.getProjectTriggers();
    triggers.forEach(trigger => ScriptApp.deleteTrigger(trigger));
    
    ScriptApp.newTrigger('sendMorningGreeting')
      .timeBased()
      .everyDays(1)
      .atHour(6)
      .create();
      
    Logger.log('สร้าง trigger สำเร็จ');
  } catch(e) {
    Logger.log('เกิดข้อผิดพลาดในการสร้าง trigger: ' + e.toString());
  }
}

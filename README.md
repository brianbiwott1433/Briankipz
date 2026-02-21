Name: BRIAN-TECH 
owner: BRIAN 
Owner:0793684491

const { Client } = require('https://chat.whatsapp.com/CiybOtSjpns0AtzFHiRet1?mode=hq1tcla.js');

const client = new Client(BRIAN);

client.on('qr', (qr) => {
    // Generate and scan this code with your phone
    console.log('QR RECEIVED', qr);
});

client.on('ready', () => {
    console.log('Client is ready!');
});

client.on('message', msg => {
    if (msg.body == '!ping') {
        msg.reply('pong');
    }
});

client.initialize();
